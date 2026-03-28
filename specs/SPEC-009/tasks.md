# SPEC-009: Visual Effects -- Tasks

## Task Order

Tasks are ordered by dependency: foundational types first, then individual effects, then pipeline integration, then keybind wiring. Each task maps to specific FRs and ACs.

---

### Task 1: FrameBuffer and FrameCell types

**FR:** Foundation for all FRs
**AC:** Prerequisite for AC-1 through AC-6

**Work:**
- Create `crates/golf-render/src/frame_buffer.rs`
- Implement `FrameBuffer` struct with `new`, `cell`, `cell_mut`, `resize`, `load_from_snapshot`
- Implement `FrameCell` struct with `age`, `fg`, `bg`, `opacity`, `flash_intensity` fields
- `load_from_snapshot` reads a `FrameSnapshot`, applies a `ColorPalette` to set initial `fg`/`bg` colors, sets `opacity = 1.0` for live cells and `0.0` for dead cells
- Unit tests: buffer creation, cell access, load from snapshot with mock palette, resize clears state

**Definition of Done:** FrameBuffer can be constructed from a FrameSnapshot with correct colors and dimensions. All tests pass.

---

### Task 2: Effect trait and EffectPipeline

**FR:** Foundation for all FRs; NFR-2 (runtime toggles)
**AC:** Prerequisite for AC-1 through AC-6

**Work:**
- Create `crates/golf-render/src/effect.rs`
- Define `Effect` trait with `apply(&mut FrameBuffer, &SimState)`, `name() -> &str`, `is_active() -> bool`
- Define `SimState` struct and `InjectionMarker` struct
- Implement `EffectPipeline` with `new`, `push`, `toggle`, `apply_all`, `list`
- `apply_all` iterates enabled effects in order, calling `apply` on each
- Unit tests: empty pipeline apply is no-op, toggle flips state, disabled effects are skipped, list returns names with states

**Definition of Done:** EffectPipeline composes effects correctly, toggle works, disabled effects are skipped. All tests pass.

---

### Task 3: SmoothTransition effect

**FR:** FR-1
**AC:** AC-1

**Work:**
- Create `crates/golf-render/src/effects/smooth.rs`
- Implement `SmoothTransition` with `prev_cells` and `transition_map`
- Delta detection: compare current cell ages against `prev_cells` to identify births and deaths
- `FadingIn` ramps opacity from 0.0 to 1.0 over `blend_frames`
- `FadingOut` ramps opacity from 1.0 to 0.0 over `blend_frames`
- Update `prev_cells` at end of each `apply` call
- Unit tests: cell birth triggers FadingIn, cell death triggers FadingOut, opacity values are correct at each frame step, stable cells have opacity 1.0, blend_frames=3 produces expected opacity sequence (0.33, 0.67, 1.0)

**Definition of Done:** AC-1 verified: a newborn cell brightens over 3 frames. Dying cells fade over 3 frames. Tests pass.

---

### Task 4: GlowEffect box filter

**FR:** FR-2
**AC:** AC-2

**Work:**
- Create `crates/golf-render/src/effects/glow.rs`
- Implement density computation: for each cell, count live neighbors in a `(2*radius+1)^2` box
- Reuse `density_buf` scratch buffer across frames (allocate once, clear each frame)
- For empty cells where density >= threshold, set `bg` to a dim palette-derived color scaled by `intensity`
- Glow color is derived from the maximum age of live cells in the box, passed through the palette
- Unit tests: isolated cell produces no glow (density below threshold), 3x3 cluster produces glow on surrounding empty cells, glow intensity scales with density, edge cells handle bounds correctly

**Definition of Done:** AC-2 verified: high-density clusters produce dim background glow on adjacent empty cells. Tests pass.

---

### Task 5: ZoomPan viewport

**FR:** FR-3
**AC:** AC-3

**Work:**
- Create `crates/golf-render/src/effects/zoom_pan.rs`
- Implement viewport calculation: `viewport_width = buf.width / zoom_level`, `viewport_height = buf.height / zoom_level`
- Nearest-neighbor sampling from grid coordinates to buffer coordinates
- Smooth interpolation: `tick()` lerps current values toward targets each frame
- `zoom_in` / `zoom_out` methods step through predefined zoom levels (1.0, 1.5, 2.0, 3.0, 4.0, 6.0, 8.0)
- `pan` adjusts target offset, clamped to grid bounds
- `reset` returns to zoom 1.0 centered
- Unit tests: zoom 2.0 maps a quarter-grid to full buffer, pan shifts viewport, out-of-bounds positions render as background, smooth interpolation converges to target over multiple ticks

**Definition of Done:** AC-3 verified: at 2x zoom, a quarter of the grid fills the terminal. Smooth scrolling works via keyboard. Tests pass.

---

### Task 6: SplitScreen pane management

**FR:** FR-4
**AC:** AC-4

**Work:**
- Create `crates/golf-render/src/effects/split_screen.rs`
- Implement `PaneLayout` enum with `Single`, `Horizontal2`, `Vertical2`, `Quad`
- `relayout` computes pane positions with 1-cell borders between panes
- `cycle_layout` advances through layouts in order
- `apply` partitions the FrameBuffer: each pane region is filled from its assigned simulation/projection
- Border cells are rendered with a dim separator character
- Unit tests: Horizontal2 produces two panes of equal width (minus border), Quad produces four panes, relayout on resize updates pane dimensions, Single layout uses full buffer

**Definition of Done:** AC-4 verified: two simulations run side by side, each in half the terminal. Layout cycling works. Tests pass.

---

### Task 7: EntropyFlash injection markers

**FR:** FR-5
**AC:** AC-5

**Work:**
- Create `crates/golf-render/src/effects/entropy_flash.rs`
- Read `SimState::entropy_events` for new injections (matching current generation)
- Create `ActiveFlash` for each new injection coordinate
- Each frame: decrement `frames_remaining`, remove expired flashes, blend flash color into `bg` at decaying intensity
- Cap `active_flashes` at `width * height` (one per cell, newest wins)
- Unit tests: new injection creates a flash, flash intensity decays linearly over decay_frames, expired flashes are removed, duplicate position replaces existing flash

**Definition of Done:** AC-5 verified: entropy injection produces a brief colored flash at injection coordinates that decays over frames. Tests pass.

---

### Task 8: AnimationSpeed frame gating

**FR:** FR-6
**AC:** AC-6

**Work:**
- Create `crates/golf-render/src/effects/animation_speed.rs`
- Implement fractional step accumulator: `should_render` adds `speed_multiplier` to `accumulated_steps`, returns true when >= 1.0
- `speed_up` / `slow_down` adjust by 0.25x, clamped to [0.25, 4.0]
- For slow-motion (multiplier < 1.0): interpolate between last-rendered and current FrameBuffer using opacity blending
- For fast-forward (multiplier > 1.0): skip visual frames, only render every Nth generation
- Unit tests: multiplier 0.5 renders every other frame, multiplier 2.0 renders every frame (simulation steps twice), multiplier 1.0 renders every frame, speed_up/slow_down clamp correctly

**Definition of Done:** AC-6 verified: animation speed 0.5x with 60 steps/sec shows 30 FPS visual output. Tests pass.

---

### Task 9: Renderer integration

**FR:** All FRs; NFR-1 (30 FPS minimum)
**AC:** All ACs (end-to-end)

**Work:**
- Add `render_from_buffer(&mut self, buf: &FrameBuffer) -> Result<()>` to `Renderer`
- Convert `FrameBuffer` cells to ratatui buffer cells: map `fg`, `bg`, `opacity` to terminal colors and characters
- Modify the render loop to use the new pipeline: `load_from_snapshot` -> `apply_all` -> `render_from_buffer`
- Keep existing `render_frame` method working (wraps the new pipeline internally)
- Add `effects` module re-exports to `crates/golf-render/src/lib.rs`
- Integration test: construct a full pipeline with all effects, render 100 frames, verify no panics and FPS stays above 30

**Definition of Done:** The render loop uses EffectPipeline. Existing rendering still works. NFR-1 verified: stacked effects maintain 30+ FPS. Integration test passes.

---

### Task 10: Runtime keybind wiring

**FR:** NFR-2 (runtime toggles); integrates with SPEC-010 FR-7
**AC:** NFR-2 verification

**Work:**
- Define keybind-to-effect mapping: `t` = SmoothTransition, `g` = Glow, `z` = ZoomPan, `e` = EntropyFlash, `s` = SplitScreen cycle, `+`/`-` = zoom, arrows = pan, `[`/`]` = speed
- Expose keybind handler from effect module: `handle_effect_key(key: KeyCode, pipeline: &mut EffectPipeline, zoom: &mut ZoomPan, speed: &mut AnimationSpeed, split: &mut SplitScreen) -> bool` (returns true if key was consumed)
- Wire into the input handling loop (coordinates with SPEC-010 keybind system)
- Manual test: run simulation, press each keybind, verify effect toggles on/off and overlay shows state change

**Definition of Done:** All effect keybinds work at runtime. Effects toggle on/off as documented. Zoom/pan/speed adjust via keyboard. NFR-2 satisfied.
