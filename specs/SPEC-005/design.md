# SPEC-005: GPU Entropy -- Design

## Signatures

### Files Created or Modified

- `crates/golf-entropy/Cargo.toml` -- add optional `cuda` feature with `cudarc` dependency
- `crates/golf-entropy/src/gpu.rs` -- `GpuEntropySource` implementation, VRAM read loop, bit mapping
- `crates/golf-entropy/src/gpu/device.rs` -- CUDA device enumeration and info reporting
- `crates/golf-entropy/src/gpu/mapping.rs` -- `BitMapping` strategies for VRAM bytes to cell coordinates
- `crates/golf-entropy/src/gpu/strategy.rs` -- `VramReadStrategy` enum and offset selection logic
- `crates/golf-entropy/src/lib.rs` -- conditional re-export of `gpu` module behind `#[cfg(feature = "cuda")]`

### Types Introduced

- `GpuEntropySource` -- implements `EntropySource`, manages CUDA device handle and read loop
  ```rust
  #[cfg(feature = "cuda")]
  pub struct GpuEntropySource {
      config: GpuEntropyConfig,
      device: Option<Arc<CudaDevice>>,
      status: SourceStatus,
      task_handle: Option<JoinHandle<()>>,
      cancel: CancellationToken,
      events_per_second: Arc<AtomicU64>,
  }
  ```

- `GpuEntropyConfig` -- all user-facing configuration for the GPU source
  ```rust
  pub struct GpuEntropyConfig {
      pub device_ordinal: usize,
      pub region_size: usize,
      pub read_interval: Duration,
      pub cells_per_read: usize,
      pub strategy: VramReadStrategy,
      pub mapping: BitMapping,
      pub grid_dimensions: Vec<usize>,
  }
  ```

- `VramReadStrategy` -- how to select the VRAM offset for each read
  ```rust
  pub enum VramReadStrategy {
      RandomWalk,
      Fixed { offset: usize },
      SequentialScan { start_offset: usize },
  }
  ```

- `BitMapping` -- how raw VRAM bytes become cell coordinates
  ```rust
  pub enum BitMapping {
      Threshold { value: u8 },
      Bitfield,
      EntropyDensity,
  }
  ```

- `CudaDeviceInfo` -- reported device metadata
  ```rust
  pub struct CudaDeviceInfo {
      pub name: String,
      pub ordinal: usize,
      pub total_memory: usize,
      pub free_memory: usize,
      pub compute_major: u32,
      pub compute_minor: u32,
  }
  ```

### Functions

- `enumerate_devices() -> Result<Vec<CudaDeviceInfo>>` -- list available CUDA devices
- `GpuEntropySource::new(config: GpuEntropyConfig) -> Result<Self>` -- initialize, attempt CUDA device creation; on failure set status to `Error(reason)` and return Ok (graceful degradation)
- `read_vram_region(device: &CudaDevice, offset: usize, size: usize) -> Result<Vec<u8>>` -- single VRAM read via `CudaSlice` device-to-host copy
- `map_bytes_to_cells(bytes: &[u8], mapping: &BitMapping, grid_dims: &[usize]) -> Vec<Vec<usize>>` -- apply the mapping strategy to produce cell coordinates
- `next_offset(strategy: &mut VramReadStrategy, current: usize, region_size: usize, total_memory: usize) -> usize` -- compute next read offset based on strategy

## Architecture

### Feature Gating

The entire GPU module is behind `#[cfg(feature = "cuda")]`. The `cuda` feature is not a default feature. When disabled, the module does not compile and no `cudarc` dependency is pulled. This keeps the default build free of CUDA runtime requirements.

```toml
# crates/golf-entropy/Cargo.toml
[features]
default = []
cuda = ["dep:cudarc"]

[dependencies]
cudarc = { version = "0.12", optional = true }
```

### VRAM Read Loop

The core loop runs inside a `tokio::task::spawn_blocking` closure (CUDA calls are synchronous and must not block the async runtime). The loop:

1. Compute the next VRAM offset using `VramReadStrategy`
2. Call `read_vram_region` to copy `region_size` bytes from device to host
3. Call `map_bytes_to_cells` to convert raw bytes into cell coordinates
4. Truncate to `cells_per_read` if the mapping produces more cells than configured
5. Send `EntropyEvent::SetCells(cells)` through the mpsc sender
6. Sleep for `read_interval`
7. Check `CancellationToken` -- break if cancelled

```
tokio::task::spawn_blocking {
    loop {
        if cancel.is_cancelled() { break; }
        let offset = next_offset(&mut strategy, current, region_size, total_mem);
        let bytes = read_vram_region(&device, offset, region_size)?;
        let cells = map_bytes_to_cells(&bytes, &mapping, &grid_dims);
        let cells = cells.into_iter().take(cells_per_read).collect();
        tx.blocking_send(EntropyEvent::SetCells(cells))?;
        std::thread::sleep(read_interval);
    }
}
```

### Bit Mapping Strategies

**Threshold**: For each byte in the VRAM buffer, if `byte > value`, the byte's index is mapped to a grid coordinate. The index is decomposed into N-dimensional coordinates using the grid dimensions (row-major index decomposition). Simple, fast, and produces variable density depending on the threshold.

**Bitfield**: Each byte contributes 8 cells. Bit position `i` of byte at index `j` maps to cell at linear index `j * 8 + i`. Produces the densest output -- 8x more cells than Threshold per byte read.

**EntropyDensity**: Bytes are consumed in chunks equal to the grid dimensionality. Each chunk is hashed (FxHash for speed, not cryptographic) to produce a coordinate. The hash output is modulo'd by each grid dimension to produce a valid coordinate. This distributes cells more uniformly across the grid regardless of VRAM content patterns.

### Linear Index to N-Dimensional Coordinate

All mappings ultimately produce a linear index that must become a `Vec<usize>` coordinate. The conversion uses row-major decomposition:

```rust
fn linear_to_nd(mut index: usize, dims: &[usize]) -> Vec<usize> {
    let mut coord = vec![0usize; dims.len()];
    for i in (0..dims.len()).rev() {
        coord[i] = index % dims[i];
        index /= dims[i];
    }
    coord
}
```

If the linear index exceeds the total grid size, it wraps via modulo (toroidal behavior, consistent with the core engine).

### Graceful Degradation

`GpuEntropySource::new` attempts to create a `CudaDevice` via `cudarc::driver::CudaDevice::new(ordinal)`. If this fails for any reason (no CUDA runtime installed, no GPU present, driver version mismatch, device ordinal out of range), the constructor does not return an error. Instead:

1. The source is created with `device: None`
2. Status is set to `SourceStatus::Error(reason_string)`
3. `start()` checks for `device.is_none()` and returns the receiver immediately (no background task spawned, no events produced)
4. `metadata()` reports the error status so the CLI and registry can display why the source is unavailable

This means the source can always be registered. The registry, CLI, and simulation engine never need CUDA-specific error handling -- they just see a source that produces zero events.

### ADR: spawn_blocking over dedicated thread

**Status:** Accepted

**Context:** CUDA runtime calls (device init, memory copy) are synchronous blocking operations. Running them on the tokio async runtime would block the executor. Two options: (a) `tokio::task::spawn_blocking` which uses tokio's blocking thread pool, (b) spawn a dedicated `std::thread` and communicate via channels.

**Decision:** Use `tokio::task::spawn_blocking`. The read loop sleeps between iterations anyway, so it naturally yields the blocking thread back to the pool during sleep intervals.

**Consequences:**
- Positive: No manual thread lifecycle management. Cancellation via `CancellationToken` integrates naturally with tokio. The blocking pool auto-scales if multiple GPU sources are active.
- Negative: `spawn_blocking` tasks share the blocking pool with other blocking work. If the pool is saturated, GPU reads could be delayed.
- Mitigation: The default blocking pool size (512 threads) is far larger than needed. GPU read interval is typically 50-100ms, so thread occupancy is low.

### ADR: cudarc over raw CUDA driver API

**Status:** Accepted

**Context:** VRAM reads require CUDA device access. Options: (a) `cudarc` crate which provides safe Rust wrappers around the CUDA driver API, (b) raw FFI bindings via `cuda-sys`, (c) writing custom `bindgen` bindings.

**Decision:** Use `cudarc`. It provides `CudaDevice` for device management and `CudaSlice<T>` for device memory with safe host-device copy operations.

**Consequences:**
- Positive: Safe abstractions, no `unsafe` blocks needed for basic memory operations. Well-maintained crate with active development. Handles CUDA context management.
- Negative: Adds a dependency on `cudarc`'s API surface. Version updates may require adaptation. The crate bundles CUDA stubs, which increases compile time when the feature is enabled.
- Mitigation: The dependency is optional (feature-gated). Pin to a compatible version range. The `cudarc` API for device memory operations is stable.
