# SPEC-006: Audio Entropy -- Design

## Signatures

### Files Created or Modified

- `crates/golf-entropy/src/audio.rs` -- `AudioEntropySource` implementing `EntropySource`
- `crates/golf-entropy/src/audio/capture.rs` -- audio device discovery and `cpal` stream management
- `crates/golf-entropy/src/audio/fft.rs` -- FFT processing pipeline using `rustfft`
- `crates/golf-entropy/src/audio/mapper.rs` -- `FrequencyBandMapper` mapping spectral data to grid coordinates
- `crates/golf-entropy/src/audio/beat.rs` -- `BeatDetector` onset detection
- `crates/golf-entropy/Cargo.toml` -- add `cpal`, `rustfft` dependencies behind `audio` feature gate

### Types Introduced

- `AudioEntropySource` -- top-level source orchestrating capture, analysis, and event emission
  ```rust
  pub struct AudioEntropySource {
      config: AudioConfig,
      status: SourceStatus,
      stream_handle: Option<cpal::Stream>,
  }
  ```
- `AudioConfig` -- user-facing configuration
  ```rust
  pub struct AudioConfig {
      pub device_name: Option<String>,  // None = default input device
      pub fft_window_size: FftWindowSize,
      pub band_mapping: BandMappingStrategy,
      pub beat_sensitivity: f64,        // 0.0..=1.0, default 0.5
      pub amplitude_floor: f32,         // below this, emit no events (silence gate)
  }
  ```
- `FftWindowSize` -- constrained FFT sizes
  ```rust
  pub enum FftWindowSize {
      W256,
      W512,
      W1024,
      W2048,
  }
  ```
- `BandMappingStrategy` -- how frequency bands map to grid regions
  ```rust
  pub enum BandMappingStrategy {
      Horizontal,  // bass=bottom, treble=top
      Vertical,    // bass=left, treble=right
      Radial,      // bass=center, treble=edges
  }
  ```
- `FrequencyBandMapper` -- maps FFT magnitude bins to grid coordinates and cell birth probabilities
  ```rust
  pub struct FrequencyBandMapper {
      grid_extents: Vec<usize>,
      strategy: BandMappingStrategy,
  }
  ```
- `BeatDetector` -- onset detection via energy spike over rolling average
  ```rust
  pub struct BeatDetector {
      energy_history: VecDeque<f32>,  // rolling window of frame energies
      history_len: usize,             // number of frames in rolling average
      sensitivity: f64,               // spike must exceed average * (1.0 + sensitivity)
  }
  ```
- `SpectralFrame` -- intermediate FFT result passed between pipeline stages
  ```rust
  pub struct SpectralFrame {
      pub magnitudes: Vec<f32>,       // magnitude per frequency bin
      pub rms_energy: f32,            // root-mean-square energy of the time-domain frame
      pub sample_rate: u32,
  }
  ```

### Functions / Hooks / Contracts

- `AudioEntropySource::new(config: AudioConfig) -> Self`
- `AudioEntropySource` implements `EntropySource` trait:
  - `fn name(&self) -> &str` -- returns `"audio"`
  - `fn kind(&self) -> &str` -- returns `"audio"`
  - `async fn start(&mut self) -> Result<Receiver<EntropyEvent>>` -- opens audio stream, spawns processing task
  - `async fn stop(&mut self) -> Result<()>` -- drops audio stream, joins processing task
  - `fn metadata(&self) -> SourceMetadata`
- `FrequencyBandMapper::new(grid_extents: Vec<usize>, strategy: BandMappingStrategy) -> Self`
- `FrequencyBandMapper::map_frame(&self, frame: &SpectralFrame) -> Vec<EntropyEvent>` -- converts spectral data to `SetCells` and `ApplyPattern` events
- `BeatDetector::new(history_len: usize, sensitivity: f64) -> Self`
- `BeatDetector::feed(&mut self, frame: &SpectralFrame) -> Option<EntropyEvent>` -- returns `Some(EnergyPulse { .. })` when a beat is detected, `None` otherwise
- `fn compute_fft(samples: &[f32], window_size: usize) -> SpectralFrame` -- runs FFT on a time-domain sample buffer, returns magnitude spectrum

## Architecture

### Processing Pipeline

Audio flows through a four-stage pipeline, all running on a single dedicated tokio blocking task spawned by `start()`:

```
cpal callback  --(ring buffer)-->  FFT stage  -->  Analysis stage  --(mpsc tx)--> SourceRegistry
                                                    |         |
                                             BeatDetector  FrequencyBandMapper
```

1. **Capture stage:** `cpal` opens the default input device (or named device). The audio callback writes f32 PCM samples into a lock-free ring buffer (`crossbeam` or `ringbuf` crate). The callback must never block or allocate.

2. **FFT stage:** The processing task reads `fft_window_size` samples from the ring buffer, applies a Hann window, runs `rustfft::FftPlanner` with a reusable scratch buffer, and produces a `SpectralFrame` with magnitude-per-bin and RMS energy.

3. **Analysis stage:** The `SpectralFrame` is fed to both the `BeatDetector` and the `FrequencyBandMapper`:
   - `BeatDetector::feed()` compares current RMS energy against the rolling average. If `current > average * (1.0 + sensitivity)`, it emits an `EnergyPulse` event centered at a position derived from the dominant frequency.
   - `FrequencyBandMapper::map_frame()` partitions the magnitude spectrum into three bands (low: 20-300 Hz, mid: 300-4000 Hz, high: 4000+ Hz), normalizes magnitudes, and maps each band to a grid region per the `BandMappingStrategy`. Bins with magnitude above the amplitude floor generate `SetCells` events.

4. **Emission:** All events from the analysis stage are sent through the `mpsc::Sender<EntropyEvent>` to the registry.

### Frequency Band Mapping

The mapper divides the grid into three regions based on the `BandMappingStrategy`:

| Strategy | Low (20-300 Hz) | Mid (300-4000 Hz) | High (4000+ Hz) |
|----------|------------------|--------------------|------------------|
| Horizontal | Bottom 33% of rows | Middle 33% | Top 33% |
| Vertical | Left 33% of columns | Middle 33% | Right 33% |
| Radial | Center 33% radius | Middle ring | Outer ring |

Within each region, individual frequency bins are mapped to specific coordinates by linear interpolation across the region's extent. The magnitude of each bin determines the cell value written (scaled to `Cell` range, i.e., 1 for any active birth).

For N-dimensional grids where N > 2, the mapper operates on the first two dimensions and ignores higher dimensions (coordinates for dimensions 2+ are set to 0). This is a deliberate simplification -- audio is inherently a 1D signal mapped to 2D; higher-dimensional mapping would require arbitrary decisions with no physical basis.

### Beat Detection Algorithm

The `BeatDetector` uses a simple onset detection approach:

1. Maintain a `VecDeque` of the last `history_len` RMS energy values (default: 43 frames, approximately 1 second at 44100 Hz / 1024 window).
2. Compute the rolling average energy.
3. If the current frame's energy exceeds `average * (1.0 + sensitivity)`, a beat is detected.
4. On detection, emit an `EnergyPulse` event:
   - `center`: derived from the dominant frequency bin -- low frequencies map to grid center, high to edges.
   - `radius`: proportional to the energy spike magnitude (bigger spike = larger pulse).
   - `intensity`: normalized spike magnitude (0.0 to 1.0).
5. After emitting, insert a cooldown of 5 frames (approximately 100ms) to avoid double-triggering on sustained transients.

### Silence Gate

When RMS energy is below `amplitude_floor` (default: 0.001), the entire analysis stage is skipped and no events are emitted. This satisfies AC-4 -- silence produces no entropy events.

### Graceful Degradation

`AudioEntropySource::start()` attempts to open the audio device. If `cpal` returns a `DevicesError` or no input devices are found:
- Set `status` to `SourceStatus::Error("no audio input device available".into())`
- Return `Err(...)` from `start()`
- The source remains registered but non-functional; the registry skips it during drain

This satisfies AC-5 and FR-6.

### Feature Gate

All audio code is behind `#[cfg(feature = "audio")]`. The `cpal` and `rustfft` dependencies are optional in `Cargo.toml`:

```toml
[features]
audio = ["dep:cpal", "dep:rustfft"]

[dependencies]
cpal = { version = "0.15", optional = true }
rustfft = { version = "6", optional = true }
```

When the `audio` feature is not enabled, `AudioEntropySource` does not exist and cannot be registered.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| `cpal` callback runs on a real-time audio thread; any blocking or allocation causes audio glitches | Garbled audio data, missed samples | Medium | The callback only writes to a lock-free ring buffer. No allocation, no locks, no channel sends in the callback path. |
| FFT processing falls behind audio capture rate | Growing ring buffer, increasing latency | Low -- FFT on 2048 samples is microseconds | Monitor ring buffer fill level. If it exceeds 4x window size, drop oldest frames (log a warning). |
| Different audio devices have different sample rates | FFT bin-to-frequency mapping is wrong | Medium | Read the device's sample rate from `cpal::SupportedStreamConfig` and pass it to `SpectralFrame`. All frequency calculations use the actual sample rate, not a hardcoded value. |
| PipeWire/PulseAudio permission issues on Linux | Source fails to open | Medium | Graceful degradation path handles this. Document that the user may need `pipewire` or `pulseaudio` access. |

## ADR-006-1: Ring Buffer over Channel for Audio Callback

**Status:** Accepted

**Context:** The `cpal` audio callback runs on a real-time thread. It must not block, allocate, or perform syscalls. `tokio::sync::mpsc::Sender::send()` can block if the channel is full, and `try_send()` still involves atomic operations that may not be suitable for real-time guarantees.

**Decision:** Use a lock-free single-producer single-consumer ring buffer (e.g., `ringbuf` crate) between the audio callback and the processing task. The callback writes samples; the processing task reads them.

**Consequences:**
- Positive: The callback path is allocation-free and lock-free. No risk of priority inversion or blocking.
- Negative: One additional dependency (`ringbuf`). The processing task must poll the ring buffer (busy-wait or sleep), adding slight latency compared to a channel's wake notification.
- Mitigation: The processing task sleeps for half the FFT window duration between polls. At 1024 samples / 44100 Hz, that is approximately 11ms sleep, well within the 20ms latency target.

## ADR-006-2: Three Fixed Bands over Configurable Band Count

**Status:** Accepted

**Context:** The frequency-to-grid mapper could support an arbitrary number of frequency bands, or use a fixed three-band (low/mid/high) split.

**Decision:** Three fixed bands with hardcoded crossover frequencies (300 Hz, 4000 Hz).

**Consequences:**
- Positive: Simple to implement, easy to reason about, maps cleanly to grid thirds. Matches how humans perceive audio (bass, midrange, treble).
- Negative: Not configurable. Users who want finer-grained frequency mapping cannot adjust band boundaries.
- Mitigation: The crossover frequencies are constants, not magic numbers buried in logic. A future enhancement could make them configurable without changing the architecture. For v1, simplicity wins.
