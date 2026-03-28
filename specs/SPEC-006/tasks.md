# SPEC-006: Audio Entropy -- Tasks

## Task List

### T1: Audio capture scaffolding
- **Scope:** Create `crates/golf-entropy/src/audio/capture.rs`. Implement device enumeration, device selection by name, and `cpal` stream setup returning a lock-free ring buffer reader. Add `cpal` and `ringbuf` as optional dependencies behind the `audio` feature gate.
- **AC coverage:** AC-5 (graceful degradation when no device), AC-6 (latency foundation)
- **Estimate:** S

### T2: FFT processing pipeline
- **Scope:** Create `crates/golf-entropy/src/audio/fft.rs`. Implement `compute_fft()` -- Hann windowing, `rustfft` forward FFT, magnitude extraction, RMS energy calculation. Return `SpectralFrame`. Add `rustfft` as optional dependency behind `audio` feature.
- **AC coverage:** AC-6 (latency -- FFT must complete in microseconds)
- **Estimate:** S

### T3: FrequencyBandMapper
- **Scope:** Create `crates/golf-entropy/src/audio/mapper.rs`. Implement `FrequencyBandMapper` with `Horizontal`, `Vertical`, and `Radial` strategies. Three fixed bands (20-300 Hz, 300-4000 Hz, 4000+ Hz). Map magnitude bins to grid coordinates and emit `SetCells` events. Clamp coordinates to grid extents.
- **AC coverage:** AC-2 (bass maps to bottom rows)
- **Estimate:** M

### T4: BeatDetector
- **Scope:** Create `crates/golf-entropy/src/audio/beat.rs`. Implement rolling average energy tracking, onset detection with configurable sensitivity, cooldown period, and `EnergyPulse` event emission.
- **AC coverage:** AC-3 (percussive beat emits EnergyPulse)
- **Estimate:** S

### T5: AudioEntropySource integration
- **Scope:** Create `crates/golf-entropy/src/audio.rs` (module root). Wire capture, FFT, mapper, and beat detector into the `AudioEntropySource` struct. Implement `EntropySource` trait. Spawn the processing task in `start()`, tear it down in `stop()`. Implement the silence gate (amplitude floor check).
- **AC coverage:** AC-1 (loud sound increases birth probability), AC-4 (silence produces no events), AC-5 (graceful degradation)
- **Estimate:** M

### T6: Unit tests
- **Scope:** Test `compute_fft()` with known sine wave input (verify peak at correct bin). Test `BeatDetector` with synthetic energy sequences (verify detection and cooldown). Test `FrequencyBandMapper` coordinate clamping and band assignment. Test `AudioEntropySource` graceful degradation with a mock device that fails.
- **AC coverage:** AC-1 through AC-6 (unit-level verification)
- **Estimate:** M

### T7: Integration test with real audio device
- **Scope:** Integration test that opens the default audio device, captures 1 second of audio, runs the full pipeline, and verifies events are produced (or gracefully handles no device in CI). Gate behind `#[cfg(feature = "audio")]` and an environment variable `GOLF_TEST_AUDIO=1` to avoid CI failures.
- **AC coverage:** AC-6 (end-to-end latency measurement)
- **Estimate:** S

## Dependency Order

```
T1 (capture) --> T2 (FFT) --> T3 (mapper) --\
                          \--> T4 (beat)  ---+--> T5 (integration) --> T6 (unit tests) --> T7 (integration test)
```

T1 and T2 are sequential (FFT reads from capture). T3 and T4 are independent of each other (both consume `SpectralFrame`). T5 wires everything together. T6 and T7 are sequential after T5.
