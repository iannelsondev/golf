# SPEC-002: Rendering — Tasks

## Task Order

Tasks are ordered by dependency: foundational types first, then rendering logic, then integration. Each task lists the FRs and ACs it addresses.

---

### Task 1: Crate Scaffold and Core Types

**FRs:** FR-1, FR-4, FR-6, FR-7
**ACs:** none directly (infrastructure)

1. Create `crates/golf-render/Cargo.toml` with dependencies: `ratatui`, `crossterm`, `golf-core` (path dependency)
2. Create `crates/golf-render/src/lib.rs` with public module declarations
3. Implement `CellChar` enum in `config.rs`
4. Implement `ProjectionMode` enum in `projection.rs`
5. Implement `RenderConfig` struct in `config.rs`
6. Add the `golf-render` crate to the workspace `Cargo.toml`

**Done when:** `cargo check -p golf-render` succeeds with all types defined.

---

### Task 2: ColorPalette Trait and Built-in Palettes

**FRs:** FR-2, FR-4
**ACs:** AC-1

1. Define `ColorPalette` trait in `palette.rs` with `color_for_age`, `trail_color`, `name`
2. Implement `Inferno` palette (black -> red -> orange -> yellow -> white gradient)
3. Implement `Ocean` palette (dark blue -> cyan -> white)
4. Implement `Matrix` palette (dark green -> bright green -> white)
5. Implement `Neon` palette (deep purple -> magenta -> pink -> white)
6. Implement `Grayscale` palette (dark gray -> white)
7. Implement `palette_by_name` lookup function
8. Write unit tests: each palette returns distinct colors for age=1 vs age=max, trail_color varies with fade

**Done when:** All five palettes pass color differentiation tests. `palette_by_name` resolves all names.

---

### Task 3: FrameState Double Buffer

**FRs:** (supports NFR-2)
**ACs:** AC-7 (enables non-blocking rendering)

1. Implement `FrameSnapshot` struct in `frame.rs`
2. Implement `FrameState` with `Arc`-swap pattern: `new`, `publish`, `read`
3. Write unit test: publish N snapshots rapidly, read returns the latest
4. Write unit test: concurrent publish and read from separate threads, no panics or data races
5. Write unit test: read returns a consistent snapshot (generation matches cell data)

**Done when:** `FrameState` passes all concurrency tests under `cargo test` and Miri (`cargo +nightly miri test` for the frame module).

---

### Task 4: N-to-2D Projection

**FRs:** FR-6
**ACs:** AC-3

1. Implement `project_to_2d` for `Flat2D` mode (identity for 2D, slice-at-zero for N>2)
2. Implement `Slice { axis, depth }` projection -- fix one dimension, extract the 2D plane
3. Implement `Flatten` projection -- max-age across higher dimensions
4. Implement `Rotate { axis, angle }` projection -- inverse rotation sampling with nearest-neighbor
5. Write unit tests for each mode with known 3D grid inputs (T-INT-002, T-INT-003)

**Done when:** All four projection modes produce correct 2D output for 3D test grids.

---

### Task 5: Death Trail System

**FRs:** FR-3
**ACs:** AC-2

1. Implement `TrailEntry` and `TrailBuffer` in `trail.rs`
2. Implement `TrailBuffer::new`, `record_deaths`, `fade_at`
3. Write unit test: cell dies at generation G, fade decreases linearly over trail_length frames (T-INT-005)
4. Write unit test: fade_at returns 0.0 after trail expires
5. Write unit test: multiple deaths at same position -- most recent dominates
6. Write unit test: trail_length=0 means fade_at always returns 0.0

**Done when:** All trail timing tests pass. Fade values are correct within f64 epsilon.

---

### Task 6: Braille Rendering

**FRs:** FR-7
**ACs:** AC-6

1. Implement `cells_to_braille` mapping function in `braille.rs` (2x4 sub-grid to U+2800-U+28FF)
2. Implement `render_braille_buffer` for full buffer conversion
3. Write unit test: all-dead sub-grid produces U+2800 (empty braille)
4. Write unit test: all-alive sub-grid produces U+28FF (full braille)
5. Write unit test: known partial pattern produces expected braille character
6. Write unit test: buffer dimensions are correctly halved/quartered (T-INT-004)

**Done when:** Braille mapping is bitwise correct for all 256 possible dot combinations.

---

### Task 7: Renderer Core and Differential Redraw

**FRs:** FR-1
**ACs:** AC-4, AC-7

1. Implement `Renderer::new` -- enter alternate screen, enable raw mode, initialize terminal
2. Implement `Renderer::render_frame` -- read FrameState, compare with prev_frame, draw only changed cells
3. Implement `Renderer::handle_resize` -- clear prev_frame, update dimensions
4. Implement `Renderer::cleanup` -- restore terminal state
5. Implement `Renderer::update_config` -- hot-reload palette, cell char, overlay toggle
6. Implement FPS tracking with rolling `VecDeque<Instant>` window
7. Write integration test: render 100 frames of a known simulation, assert no panics (T-INT-006 for resize)
8. Write benchmark: measure frame render time for 200x50 grid, assert < 16ms (AC-7)

**Done when:** Renderer produces visible output in a terminal. Benchmark confirms < 16ms per frame for 200x50.

---

### Task 8: Stats Overlay

**FRs:** FR-5
**ACs:** AC-5

1. Implement `OverlayStats` struct in `overlay.rs`
2. Implement `Renderer::render_overlay` -- draw stats block in top-right corner
3. Display: FPS, generation count, live cell count, dimensions, rule description
4. Write unit test: overlay content includes all required fields when show_overlay=true
5. Write unit test: overlay is not rendered when show_overlay=false

**Done when:** Overlay displays all five stats fields. Toggling show_overlay on/off works within one frame.

---

### Task 9: End-to-End Integration

**FRs:** all
**ACs:** AC-1 through AC-7

1. Write integration test: full pipeline from `SimulationEngine<2>` through `project_to_2d` through `FrameState` through `Renderer` (T-INT-001)
2. Write integration test: 3D simulation with slice mode, change depth, verify output changes (AC-3)
3. Write integration test: frame drop under load -- 100 rapid publishes, one read (T-INT-007)
4. Write integration test: palette color mapping for all built-ins (T-INT-008)
5. Run full benchmark suite: 200x50 grid, 60 seconds, measure average FPS and P99 frame time

**Done when:** All integration tests pass. Benchmark confirms sustained 60 FPS for 200x50 2D grid.
