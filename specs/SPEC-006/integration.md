# SPEC-006: Audio Entropy -- Integration

## Consumed Interfaces

### SPEC-003 (Entropy Trait)
- Consumes: `EntropySource` trait, `EntropyEvent` enum, `SourceStatus`, `SourceMetadata`
- Version: SPEC-003 as of main branch at implementation time
- Contract: `AudioEntropySource` implements `EntropySource`. Calls to `start()` return a `Receiver<EntropyEvent>`. Events emitted are `SetCells(Vec<Vec<usize>>)`, `ApplyPattern { cells, offset }`, and `EnergyPulse { center, radius, intensity }`.

### SPEC-001 (Core Engine)
- Consumes: Grid extents for coordinate mapping
- Version: SPEC-001 as of main branch at implementation time
- Contract: `FrequencyBandMapper` requires the grid's `extents` (as `Vec<usize>`) to map frequency bins to valid coordinates. Extents are read once at source construction and cached. If the grid is resized at runtime, the mapper must be reconstructed.

## Provided Interfaces

### To SPEC-003 (Entropy Trait) / SourceRegistry
- Provides: `AudioEntropySource` as a registerable `Box<dyn EntropySource>`
- Registration: The CLI or config layer constructs an `AudioConfig`, creates an `AudioEntropySource`, and registers it with the `SourceRegistry`. The registry manages its lifecycle.

### To SPEC-010 (CLI & Config)
- Provides: `AudioConfig` struct for CLI flag and TOML config deserialization
- Contract: `AudioConfig` derives `serde::Deserialize` and `clap::Args` (or equivalent) so the CLI can parse `--audio-device`, `--fft-window`, `--beat-sensitivity`, and `--band-mapping` flags.

## Integration Points

### Audio Device Selection
The CLI passes an optional device name string. `AudioEntropySource` uses `cpal::default_host().input_devices()` to enumerate devices. If `device_name` is `Some(name)`, it searches for a device whose name contains the given string (case-insensitive substring match). If `None`, it uses `default_input_device()`.

### Coordinate System
`FrequencyBandMapper` produces coordinates as `Vec<usize>` matching the SPEC-003 `EntropyEvent` coordinate convention. The mapper clamps all coordinates to `[0, extent-1]` per dimension to guarantee the simulation engine never receives out-of-bounds coordinates.

### Feature Gate Boundary
When `audio` feature is disabled, the `AudioEntropySource` type does not exist. Code that conditionally registers the audio source must use `#[cfg(feature = "audio")]` guards. The CLI should list available sources based on compiled features.

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| No audio input device | `cpal` returns empty device list or `default_input_device()` returns `None` | `start()` returns `Err`, source status set to `Error`. Registry skips during drain. |
| Audio device disconnected mid-stream | `cpal` stream error callback fires | Set status to `Error`, stop the processing task. Log a warning. Source can be restarted if device reappears. |
| Ring buffer overflow (processing too slow) | Ring buffer write returns `Err` (full) | Drop oldest samples. Log a warning at Debug level (expected during transient load spikes). |
| FFT produces NaN/Inf magnitudes | Check `f32::is_finite()` on magnitude values | Replace non-finite values with 0.0. Log a warning at Debug level. |
