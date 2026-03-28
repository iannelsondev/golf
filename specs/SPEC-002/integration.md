# SPEC-002: Rendering — Integration

## Interfaces Provided

### Renderer

- **Type:** `Renderer` struct in `crates/golf-render/src/renderer.rs`
- **Contract:** Owns the terminal. Accepts `FrameState` references and renders frames at up to 60 FPS. Handles resize, overlay toggle, config hot-reload, and terminal cleanup on drop.
- **Consumers:** SPEC-010 (CLI -- creates and owns the Renderer), SPEC-009 (Visual Effects -- extends rendering pipeline)

### ColorPalette

- **Type:** `trait ColorPalette` in `crates/golf-render/src/palette.rs`
- **Contract:** Maps cell age to RGB color and dead-cell fade to trail color. Implementations must be `Send + Sync`.
- **Built-in implementations:** `Inferno`, `Ocean`, `Matrix`, `Neon`, `Grayscale`
- **Consumers:** SPEC-009 (Visual Effects -- may add custom palettes), SPEC-010 (CLI -- palette selection by name)

### ProjectionMode

- **Type:** `enum ProjectionMode` in `crates/golf-render/src/projection.rs`
- **Contract:** Defines how N-dimensional grids are projected to 2D. Variants: `Flat2D`, `Slice { axis, depth }`, `Flatten`, `Rotate { axis, angle }`.
- **Consumers:** SPEC-010 (CLI -- projection mode selection), SPEC-009 (Visual Effects -- animated projection transitions)

### FrameState

- **Type:** `FrameState` struct in `crates/golf-render/src/frame.rs`
- **Contract:** Thread-safe double buffer. Simulation thread calls `publish(snapshot)`, renderer thread calls `read()`. The renderer never blocks the simulation. Intermediate frames are silently dropped.
- **Consumers:** SPEC-001 (Core Engine -- publishes frames after each step), SPEC-010 (CLI -- wires sim thread to render thread)

### project_to_2d

- **Type:** `fn project_to_2d<const N: usize>(grid: &NdGrid<N>, mode: &ProjectionMode, width: usize, height: usize) -> Vec<Cell>`
- **Contract:** Pure function. Projects N-dimensional grid state to a flat 2D cell buffer for rendering.
- **Consumers:** The simulation loop (or a bridge layer) calls this before publishing to FrameState.

## Interfaces Consumed

### SPEC-001 (Core Engine)

- **Version:** SPEC-001 design.md as of initial commit (pre-v1)
- **Consumed types:**
  - `Cell` (`u16`) -- used throughout rendering to determine age-based color and alive/dead state
  - `NdGrid<N>` -- read via `grid.cells()` (flat slice), `grid.extents()` (dimensions), `grid.len()` (total count)
  - `SimulationEngine<N>` -- read via `engine.grid()`, `engine.generation()`, `engine.rules()` to populate `FrameSnapshot`
  - `RuleSet` -- read `birth`, `survival`, `neighborhood` fields to build `rule_description` string for overlay
- **Contract:** SPEC-002 reads grid state after each simulation step. It never mutates the grid. The `Cell` type must remain `u16` with 0=dead, >0=age semantics. `NdGrid::cells()` must return a flat row-major slice.
- **Breaking change sensitivity:** If `Cell` changes from `u16` to a different type, all palette and trail logic must be updated. If `NdGrid::cells()` changes its layout from row-major, projection and braille mapping break.

## Event Flows

### Frame Production (Simulation to Renderer)

```
SimulationEngine::step()
  -> project_to_2d(engine.grid(), projection_mode, width, height)
  -> construct FrameSnapshot { cells, width, height, generation, live_count, dimensions, rule_description }
  -> FrameState::publish(snapshot)
  -- (renderer thread) --
  -> FrameState::read()
  -> Renderer::render_frame()
    -> compare with prev_frame (differential redraw)
    -> TrailBuffer::record_deaths(prev, current, generation)
    -> for each cell: palette.color_for_age() or trail.fade_at() + palette.trail_color()
    -> if braille mode: cells_to_braille() for each 2x4 sub-grid
    -> if show_overlay: render_overlay() with OverlayStats
    -> terminal.draw()
    -> update prev_frame
```

### Terminal Resize

```
crossterm::event::Event::Resize(width, height)
  -> Renderer::handle_resize(width, height)
    -> clear prev_frame (force full redraw)
    -> update TrailBuffer dimensions
    -> signal simulation to update projection dimensions (via shared config or channel)
  -> next render_frame() redraws everything
```

### Configuration Hot-Reload

```
User keypress (e.g., 'p' for palette cycle, 'o' for overlay toggle)
  -> Renderer::update_config(new_config)
    -> clear prev_frame if palette changed (colors need full redraw)
    -> update trail capacity if trail_length changed
  -> next render_frame() uses new config
```

## Integration Test Requirements

### T-INT-001: Sim-to-Renderer Frame Flow

Verify that a `SimulationEngine<2>` stepping a 10x10 grid produces frames that `FrameState` delivers to a mock renderer consumer. Assert:
- `FrameSnapshot.cells` length equals width * height
- `FrameSnapshot.generation` increments
- `FrameSnapshot.live_count` matches actual alive cells in the buffer

### T-INT-002: Projection Correctness (Slice Mode)

Create a `NdGrid<3>` with dimensions [10, 10, 5]. Set known cells at specific Z depths. Call `project_to_2d` with `Slice { axis: 2, depth: d }` for each depth. Assert that the output 2D buffer contains exactly the cells at that Z layer.

### T-INT-003: Projection Correctness (Flatten Mode)

Create a `NdGrid<3>` with overlapping live cells at different Z layers. Call `project_to_2d` with `Flatten`. Assert that each 2D position contains the maximum age across all Z layers.

### T-INT-004: Braille Density

Create a 4x8 cell buffer with a known pattern. Call `render_braille_buffer`. Assert output dimensions are 2x2 (4/2 x 8/4). Assert braille characters match expected bitmask for the pattern.

### T-INT-005: Trail Fade Timing

Simulate a cell dying at generation G. Call `TrailBuffer::fade_at` at generations G+1 through G+trail_length. Assert fade values decrease linearly from near 1.0 to 0.0. Assert fade_at returns 0.0 at G+trail_length+1.

### T-INT-006: Resize Recovery

Publish a frame to `FrameState`, call `handle_resize` with new dimensions, then call `render_frame`. Assert no panic and the rendered output matches the new terminal dimensions.

### T-INT-007: Frame Drop Under Load

Publish 100 frames to `FrameState` rapidly (simulating a fast simulation). Call `read()` once. Assert the returned snapshot is the most recently published one (generation == 100), confirming intermediate frames were dropped.

### T-INT-008: Palette Color Mapping

For each built-in palette, call `color_for_age(1, 100)` and `color_for_age(100, 100)`. Assert the colors are different (young and old cells are visually distinct). Call `trail_color(1.0)` and `trail_color(0.1)`. Assert both return valid colors and they differ.
