# SPEC-005: GPU Entropy -- Tasks

## Task Order

Tasks are ordered by dependency. Each task produces a testable artifact.

### Task 1: Feature gate and dependency setup

Add the `cuda` optional feature and `cudarc` dependency to `crates/golf-entropy/Cargo.toml`. Add the conditional `pub mod gpu` to `lib.rs` behind `#[cfg(feature = "cuda")]`. Verify the crate compiles with and without the `cuda` feature.

**Acceptance criteria covered:** None directly (infrastructure).

### Task 2: CUDA device enumeration

Implement `CudaDeviceInfo` struct and `enumerate_devices()` function in `gpu/device.rs`. Use `cudarc::driver::CudaDevice::new(ordinal)` to probe available devices and report name, memory, and compute capability.

**Tests:**
- On a CUDA system: `enumerate_devices` returns at least one device with non-zero memory
- On a non-CUDA system: `enumerate_devices` returns an error (not a panic)

**Acceptance criteria covered:** AC-1

### Task 3: VRAM read function

Implement `read_vram_region(device, offset, size) -> Result<Vec<u8>>`. Allocate a `CudaSlice<u8>` at the given device offset, copy to host. Handle out-of-bounds offsets by clamping to available memory.

**Tests:**
- Read a 64KB region and verify the returned buffer length matches the requested size
- Read with an offset beyond device memory returns an error

**Acceptance criteria covered:** AC-2 (partial), AC-6 (verified by NFR-1 timing)

### Task 4: Bit mapping strategies

Implement `BitMapping` enum and `map_bytes_to_cells` function in `gpu/mapping.rs`. Implement the `linear_to_nd` helper for coordinate decomposition.

**Tests (table-driven):**
- Threshold: bytes `[0, 128, 255]` with threshold 127 produce cells for indices 1 and 2 only
- Bitfield: byte `0b10100000` produces cells at bit positions 5 and 7
- EntropyDensity: given known input bytes and grid dims, output coordinates are within grid bounds
- `linear_to_nd`: index 5 in a 2x3 grid produces `[1, 2]`

**Acceptance criteria covered:** AC-2

### Task 5: VRAM read strategy and offset selection

Implement `VramReadStrategy` enum and `next_offset` function in `gpu/strategy.rs`.

**Tests:**
- RandomWalk: two consecutive calls produce different offsets (statistical: run 10 times, at least 8 differ)
- Fixed: always returns the configured offset
- SequentialScan: advances by `region_size` each call, wraps at `total_memory`

**Acceptance criteria covered:** AC-3

### Task 6: GpuEntropySource struct and EntropySource impl

Implement `GpuEntropySource` with `new`, `start`, `stop`, `name`, `kind`, `metadata`. Wire up the VRAM read loop inside `start` using `tokio::task::spawn_blocking`. Use `CancellationToken` for shutdown.

**Tests:**
- `new` with invalid device ordinal succeeds but sets status to `Error`
- `start` on a degraded source returns a receiver that produces no events
- `name` returns `"gpu-vram"`
- `kind` returns `"cuda"`

**Acceptance criteria covered:** AC-4

### Task 7: Rate control and injection integration

Wire `cells_per_read` truncation and `read_interval` timing into the read loop. Verify that the source respects the configured injection rate.

**Tests:**
- Configure 100 cells/read at 10 reads/sec, run for 1 second, count received cells (expect approximately 1000, tolerance +/-20%)
- Configure 50ms read interval, measure time between events (expect approximately 50ms, tolerance +/-15ms)

**Acceptance criteria covered:** AC-5

### Task 8: Integration with SourceRegistry

Register `GpuEntropySource` in a `SourceRegistry`, start it, call `drain`, and verify events flow through. Test the full lifecycle: register, start, drain events, stop, unregister.

**Tests:**
- Registry lists the GPU source with correct name and kind
- `drain` returns `SetCells` events when the source is running
- After `stop`, no further events are produced

**Acceptance criteria covered:** AC-1 through AC-5 (end-to-end)
