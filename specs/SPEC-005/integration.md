# SPEC-005: GPU Entropy -- Integration

## Consumed Interfaces

### SPEC-003 (Entropy Trait)

- **Consumes:** `EntropySource` trait, `EntropyEvent` enum, `SourceStatus` enum, `SourceMetadata` struct
- **Version:** SPEC-003 as of main branch at implementation time
- **Contract:** `GpuEntropySource` implements `EntropySource` with the following behavior:
  - `name()` returns `"gpu-vram"` (or `"gpu-vram-{ordinal}"` when multiple devices are active)
  - `kind()` returns `"cuda"`
  - `start()` spawns the VRAM read loop and returns a `Receiver<EntropyEvent>`; if no CUDA device is available, returns an immediately-drainable receiver (no events)
  - `stop()` cancels the read loop via `CancellationToken` and awaits task completion
  - `metadata()` reports current `SourceStatus` and observed `events_per_second`
- **Events produced:** `EntropyEvent::SetCells(Vec<Vec<usize>>)` exclusively. The GPU source does not produce `ClearRegion`, `ApplyPattern`, `MutateRules`, or `EnergyPulse` events.

### SPEC-003 (SourceRegistry)

- **Consumes:** `SourceRegistry::register(source: Box<dyn EntropySource>) -> SourceId`
- **Version:** SPEC-003 as of main branch at implementation time
- **Contract:** `GpuEntropySource` is registered as a `Box<dyn EntropySource>`. The registry manages its lifecycle (start/stop) and drains its events. No GPU-specific code exists in the registry.

### SPEC-001 (Core Engine)

- **Consumes:** Grid dimension information (the `grid_dimensions: Vec<usize>` passed to `GpuEntropyConfig`)
- **Version:** SPEC-001 as of main branch at implementation time
- **Contract:** The GPU source produces coordinates that are valid for the target grid dimensions. Coordinate validity is the source's responsibility (modulo wrapping via `linear_to_nd`). The core engine validates coordinate dimensionality when applying events but should not encounter mismatches from this source.

## Provided Interfaces

### GpuEntropySource

- **Provided to:** Any consumer that registers entropy sources (CLI, config system, test harness)
- **Public API:**
  - `GpuEntropySource::new(config: GpuEntropyConfig) -> Result<Self>` -- constructor, always succeeds (graceful degradation)
  - `enumerate_devices() -> Result<Vec<CudaDeviceInfo>>` -- standalone function for device discovery
  - Trait methods via `EntropySource` implementation
- **Feature gate:** All types and functions are behind `#[cfg(feature = "cuda")]`

### CudaDeviceInfo

- **Provided to:** CLI (`golf list-sources`), config validation
- **Contract:** Reports device name, ordinal, total/free memory, and compute capability. Used for display and to validate `device_ordinal` in config.

## Integration Points

### Cargo Feature Wiring

The `golf-cli` crate (or top-level binary crate) must forward the `cuda` feature to `golf-entropy`:

```toml
# crates/golf-cli/Cargo.toml
[features]
cuda = ["golf-entropy/cuda"]

[dependencies]
golf-entropy = { path = "../golf-entropy" }
```

The CLI conditionally registers the GPU source:

```rust
#[cfg(feature = "cuda")]
{
    match GpuEntropySource::new(gpu_config) {
        Ok(source) => { registry.register(Box::new(source)); }
        Err(e) => { tracing::warn!("GPU entropy source unavailable: {e}"); }
    }
}
```

### Channel Capacity

The GPU source creates its mpsc channel with capacity `max_events_per_drain * 4` (matching the SPEC-003 convention). At the default rate of 10 reads/second, this provides 400ms of buffering before backpressure.

### Coordinate Dimensionality

The GPU source receives `grid_dimensions` at construction time and produces coordinates matching those dimensions. If the simulation changes grid dimensions at runtime (not currently supported but possible in future specs), the source must be stopped and restarted with updated dimensions.
