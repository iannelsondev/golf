# SPEC-009: Visual Effects -- Design

## Signatures

### Files Created or Modified

- `crates/golf-render/src/effect.rs` -- `Effect` trait, `EffectPipeline`, effect ordering
- `crates/golf-render/src/effects/mod.rs` -- module root for effect implementations
- `crates/golf-render/src/effects/smooth.rs` -- `SmoothTransition` effect (FR-1)
- `crates/golf-render/src/effects/glow.rs` -- `GlowEffect` box-filter blur (FR-2)
- `crates/golf-render/src/effects/zoom_pan.rs` -- `ZoomPan` viewport effect (FR-3)
- `crates/golf-render/src/effects/split_screen.rs` -- `SplitScreen` pane manager (FR-4)
- `crates/golf-render/src/effects/entropy_flash.rs` -- `EntropyFlash` injection marker (FR-5)
- `crates/golf-render/src/effects/animation_speed.rs` -- `AnimationSpeed` frame rate decoupling (FR-6)
- `crates/golf-render/src/frame_buffer.rs` -- `FrameBuffer` intermediate 2D cell+color buffer for effect composition
- `crates/golf-render/src/renderer.rs` -- modified to integrate `EffectPipeline` into the render loop

### Types Introduced

```rust
/// Intermediate buffer that effects read and write.
/// Decouples effect processing from terminal output.
pub struct FrameBuffer {
    pub cells: Vec<FrameCell>,
    pub width: usize,
    pub height: usize,
}

/// A single cell in the FrameBuffer with both logical state and visual output.
pub struct FrameCell {
    pub age: u16,             // from simulation (0 = dead)
    pub fg: Color,            // foreground color (computed by palette, modified by effects)
    pub bg: Color,            // background color (modified by glow, flash, etc.)
    pub opacity: f32,         // 0.0 = fully transparent, 1.0 = fully opaque
    pub flash_intensity: f32, // entropy flash overlay, decays per frame
}

/// Snapshot of simulation state passed to effects for read-only queries.
pub struct SimState<'a> {
    pub cells: &'a [u16],
    pub width: usize,
    pub height: usize,
    pub generation: u64,
    pub live_count: u64,
    pub entropy_events: &'a [InjectionMarker],
}

/// Records where an entropy event was injected, for visualization.
pub struct InjectionMarker {
    pub x: usize,
    pub y: usize,
    pub generation: u64,     // when the injection happened
    pub source_name: String, // which entropy source produced it
}

/// Ordered, composable chain of effects applied each frame.
pub struct EffectPipeline {
    effects: Vec<Box<dyn Effect>>,
    enabled: Vec<bool>,       // per-effect toggle (indexed parallel to effects)
}

/// Smooth birth/death transitions over multiple frames.
pub struct SmoothTransition {
    pub blend_frames: u8,     // number of frames to interpolate over (default 3)
    prev_cells: Vec<u16>,     // previous frame's cell state for delta detection
    transition_map: Vec<TransitionState>,
}

/// Per-cell transition tracking.
enum TransitionState {
    Stable,
    FadingIn { frames_remaining: u8 },
    FadingOut { frames_remaining: u8 },
}

/// Box-filter glow around high-density cell regions.
pub struct GlowEffect {
    pub radius: u8,           // blur radius in cells (default 2)
    pub threshold: u8,        // minimum live neighbors to trigger glow (default 3)
    pub intensity: f32,       // glow brightness multiplier (default 0.4)
    density_buf: Vec<u8>,     // scratch buffer for neighbor density counts
}

/// Keyboard-driven zoom and pan viewport.
pub struct ZoomPan {
    pub zoom_level: f32,      // 1.0 = no zoom, 2.0 = 2x, max 8.0
    pub offset_x: f32,        // viewport center X in grid coordinates
    pub offset_y: f32,        // viewport center Y in grid coordinates
    pub smooth_factor: f32,   // interpolation speed for smooth scrolling (0.0-1.0)
    target_zoom: f32,         // target zoom for smooth interpolation
    target_x: f32,
    target_y: f32,
}

/// Split-screen pane layout.
pub struct SplitScreen {
    pub layout: PaneLayout,
    pub panes: Vec<Pane>,
}

pub enum PaneLayout {
    Single,                   // no split
    Horizontal2,              // 2 panes side by side
    Vertical2,                // 2 panes stacked
    Quad,                     // 2x2 grid
}

/// One pane in a split-screen layout.
pub struct Pane {
    pub x: usize,
    pub y: usize,
    pub width: usize,
    pub height: usize,
    pub sim_index: usize,     // which simulation to render (index into a sim registry)
    pub projection: ProjectionMode,
}

/// Brief colored flash at entropy injection coordinates.
pub struct EntropyFlash {
    pub decay_frames: u8,     // frames before flash fully fades (default 5)
    pub color: Color,         // base flash color (default bright yellow)
    active_flashes: Vec<ActiveFlash>,
}

struct ActiveFlash {
    x: usize,
    y: usize,
    frames_remaining: u8,
}

/// Decouples visual framerate from simulation step rate.
pub struct AnimationSpeed {
    pub speed_multiplier: f32,    // 0.25 = quarter speed, 1.0 = normal, 4.0 = 4x
    accumulated_steps: f32,       // fractional step accumulator
    last_rendered_generation: u64,
}
```

### Functions / Traits / Contracts

```rust
// --- Effect trait ---

/// All visual effects implement this trait. Effects are composable and order-dependent:
/// earlier effects in the pipeline write to the FrameBuffer first, later effects
/// read and modify what earlier effects produced.
pub trait Effect: Send + Sync {
    /// Apply this effect to the frame buffer, using simulation state for context.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);

    /// Human-readable name for display in overlay and keybind listing.
    fn name(&self) -> &str;

    /// Whether this effect is currently doing work. Effects that have no active
    /// transitions or flashes can return false to allow the pipeline to skip them.
    fn is_active(&self) -> bool { true }
}

// --- FrameBuffer ---

impl FrameBuffer {
    /// Allocate a new frame buffer of the given dimensions.
    pub fn new(width: usize, height: usize) -> Self;

    /// Get a mutable reference to the cell at (x, y).
    pub fn cell_mut(&mut self, x: usize, y: usize) -> &mut FrameCell;

    /// Get a reference to the cell at (x, y).
    pub fn cell(&self, x: usize, y: usize) -> &FrameCell;

    /// Fill the buffer from a FrameSnapshot, applying the palette for initial colors.
    pub fn load_from_snapshot(
        &mut self,
        snapshot: &FrameSnapshot,
        palette: &dyn ColorPalette,
    );

    /// Resize the buffer (e.g., on terminal resize). Clears all cell state.
    pub fn resize(&mut self, width: usize, height: usize);
}

// --- EffectPipeline ---

impl EffectPipeline {
    /// Create an empty pipeline.
    pub fn new() -> Self;

    /// Add an effect to the end of the pipeline. Returns its index for toggling.
    pub fn push(&mut self, effect: Box<dyn Effect>) -> usize;

    /// Toggle the effect at the given index on/off. Returns the new state.
    pub fn toggle(&mut self, index: usize) -> bool;

    /// Apply all enabled effects in order.
    pub fn apply_all(&mut self, buf: &mut FrameBuffer, sim: &SimState);

    /// List all effects with their enabled/disabled state.
    pub fn list(&self) -> Vec<(&str, bool)>;
}

// --- SmoothTransition ---

impl SmoothTransition {
    pub fn new(blend_frames: u8) -> Self;
}

impl Effect for SmoothTransition {
    /// Detects cells that changed state since the previous frame.
    /// For births: ramps opacity from 0.0 to 1.0 over blend_frames.
    /// For deaths: ramps opacity from 1.0 to 0.0 over blend_frames.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Smooth Transitions" }
}

// --- GlowEffect ---

impl GlowEffect {
    pub fn new(radius: u8, threshold: u8, intensity: f32) -> Self;
}

impl Effect for GlowEffect {
    /// Computes a density map by counting live neighbors in a box of `radius`
    /// around each cell. Where density exceeds `threshold`, applies a dim
    /// background color bleed to neighboring empty cells. The glow color is
    /// derived from the dominant age in the high-density region via the palette.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Glow" }
}

// --- ZoomPan ---

impl ZoomPan {
    pub fn new() -> Self;

    /// Zoom in by one step (1.0 -> 1.5 -> 2.0 -> 3.0 -> 4.0 -> 6.0 -> 8.0).
    pub fn zoom_in(&mut self);

    /// Zoom out by one step (reverse of zoom_in). Clamps at 1.0.
    pub fn zoom_out(&mut self);

    /// Pan by the given delta in grid-space cells.
    pub fn pan(&mut self, dx: f32, dy: f32);

    /// Reset to no zoom, centered viewport.
    pub fn reset(&mut self);

    /// Advance smooth interpolation one frame tick. Call once per render frame.
    pub fn tick(&mut self);
}

impl Effect for ZoomPan {
    /// Remaps the FrameBuffer to show only the viewport region at the current
    /// zoom level and offset. Cells outside the viewport are not rendered.
    /// Uses nearest-neighbor sampling (sufficient for terminal character cells).
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Zoom/Pan" }
}

// --- SplitScreen ---

impl SplitScreen {
    pub fn new(layout: PaneLayout) -> Self;

    /// Recalculate pane positions for the given terminal dimensions.
    pub fn relayout(&mut self, term_width: usize, term_height: usize);

    /// Cycle to the next layout: Single -> H2 -> V2 -> Quad -> Single.
    pub fn cycle_layout(&mut self, term_width: usize, term_height: usize);
}

impl Effect for SplitScreen {
    /// Divides the FrameBuffer into pane regions. Each pane renders its
    /// assigned simulation/projection independently. Draws 1-cell borders
    /// between panes. This effect MUST be applied last in the pipeline
    /// because it restructures the buffer layout.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Split Screen" }
}

// --- EntropyFlash ---

impl EntropyFlash {
    pub fn new(decay_frames: u8, color: Color) -> Self;
}

impl Effect for EntropyFlash {
    /// Reads entropy_events from SimState. For each new injection, creates an
    /// ActiveFlash. For each active flash, blends the flash color into the
    /// cell's background at an intensity proportional to frames_remaining / decay_frames.
    /// Removes expired flashes.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Entropy Flash" }
    fn is_active(&self) -> bool { !self.active_flashes.is_empty() }
}

// --- AnimationSpeed ---

impl AnimationSpeed {
    pub fn new(speed_multiplier: f32) -> Self;

    /// Increase speed by 0.25x (clamped to 4.0).
    pub fn speed_up(&mut self);

    /// Decrease speed by 0.25x (clamped to 0.25).
    pub fn slow_down(&mut self);

    /// Given the current simulation generation, returns true if a visual
    /// frame should be rendered this tick.
    pub fn should_render(&mut self, current_generation: u64) -> bool;
}

impl Effect for AnimationSpeed {
    /// When speed_multiplier < 1.0: interpolates between frames to show
    /// smooth slow-motion. When > 1.0: skips intermediate visual frames,
    /// only rendering every Nth generation.
    fn apply(&mut self, buf: &mut FrameBuffer, sim: &SimState);
    fn name(&self) -> &str { "Animation Speed" }
}
```

## Architecture

### FrameBuffer as Effect Intermediate

Effects do not write directly to the terminal. Instead, the render loop introduces a `FrameBuffer` between the `FrameState` (simulation output) and the `Renderer` (terminal output):

```
FrameState::read()
    |
    v
FrameBuffer::load_from_snapshot()   -- palette applied, initial colors set
    |
    v
EffectPipeline::apply_all()          -- each enabled effect mutates the FrameBuffer
    |
    v
Renderer::render_from_buffer()       -- writes FrameBuffer to terminal via ratatui
```

This indirection keeps effects composable. Each effect reads the current FrameBuffer state (which may include modifications from earlier effects) and writes its changes back. The Renderer only needs to know how to paint a `FrameBuffer` -- it does not know about individual effects.

The `FrameBuffer` is allocated once and reused across frames. On terminal resize, it is resized in place. This avoids per-frame allocation.

### Effect Ordering

Effects are applied in pipeline order. Order matters because later effects see earlier effects' changes. The default ordering is:

1. `AnimationSpeed` -- decides whether this frame renders at all
2. `SmoothTransition` -- modifies cell opacity for birth/death blending
3. `GlowEffect` -- reads cell density, writes background colors
4. `EntropyFlash` -- overlays flash markers on top of glow
5. `ZoomPan` -- remaps the buffer to the viewport (must happen after content effects)
6. `SplitScreen` -- restructures the buffer into panes (must be last)

The pipeline does not enforce this ordering -- it applies effects in insertion order. The default pipeline constructor sets this order. Users who build custom pipelines via the CLI or config can reorder, but the documentation warns that ZoomPan and SplitScreen must come after content-modifying effects.

### SmoothTransition Delta Detection

`SmoothTransition` maintains a copy of the previous frame's cell ages (`prev_cells`). Each frame, it compares current vs previous:

- Cell was dead, now alive: begin `FadingIn` with `frames_remaining = blend_frames`.
- Cell was alive, now dead: begin `FadingOut` with `frames_remaining = blend_frames`.
- Otherwise: `Stable` (no change).

During `FadingIn`, the cell's `opacity` in the FrameBuffer is set to `1.0 - (frames_remaining as f32 / blend_frames as f32)`. During `FadingOut`, opacity is `frames_remaining as f32 / blend_frames as f32`. Each frame, `frames_remaining` decrements by 1. When it reaches 0, the transition state resets to `Stable`.

This satisfies AC-1: a newborn cell brightens over `blend_frames` (default 3) frames.

### GlowEffect Box Filter

The glow effect uses a box filter to compute local cell density:

1. Allocate (or reuse) a scratch buffer `density_buf` of size `width * height`.
2. For each cell position, count live cells in a `(2*radius+1) x (2*radius+1)` box centered on that position. Store the count in `density_buf`.
3. For each empty cell where `density_buf[pos] >= threshold`, set the cell's `bg` color to a dim version of the dominant live-cell color in the box. The intensity is `(density / max_possible_density) * self.intensity`.

The box filter is O(width * height * radius^2). For typical terminal sizes (200x50) and radius 2, this is 200 * 50 * 25 = 250,000 operations per frame -- well within the NFR-1 budget of maintaining 30+ FPS.

This satisfies AC-2: adjacent empty cells near high-density clusters show a dim background glow.

### ZoomPan Viewport Mapping

ZoomPan works by defining a viewport rectangle in grid-space coordinates:

```
viewport_width  = buffer_width  / zoom_level
viewport_height = buffer_height / zoom_level
viewport_x      = offset_x - viewport_width  / 2
viewport_y      = offset_y - viewport_height / 2
```

During `apply`, the effect remaps each FrameBuffer cell `(bx, by)` to the grid-space coordinate:

```
gx = viewport_x + bx / zoom_level
gy = viewport_y + by / zoom_level
```

If `(gx, gy)` is within the grid bounds, the cell at that position is sampled. Otherwise, the cell is set to the background color. Nearest-neighbor sampling is used (truncate to integer coordinates).

Smooth scrolling is achieved by interpolating `(offset_x, offset_y, zoom_level)` toward their target values each frame using `smooth_factor` as a lerp weight. Keyboard events set the targets; `tick()` moves current values toward targets.

At zoom 2.0, a quarter of the grid fills the terminal -- satisfying AC-3.

### SplitScreen Pane Management

SplitScreen divides the terminal into rectangular panes. Each pane gets its own sub-region of the FrameBuffer. The `relayout` method computes pane positions based on terminal dimensions and the active `PaneLayout`:

- `Horizontal2`: two panes, each `width/2` wide, full height. 1-cell vertical border between them.
- `Vertical2`: two panes, each `height/2` tall, full width. 1-cell horizontal border.
- `Quad`: four panes in a 2x2 grid.

Each `Pane` references a simulation index. In the simplest case (single simulation), different panes show different projection modes of the same simulation. When multiple simulations are running (configured via CLI), each pane can show a different simulation.

SplitScreen must be the last effect in the pipeline because it restructures which simulation data maps to which buffer region. Effects applied before SplitScreen operate on the full buffer; SplitScreen then partitions it.

This satisfies AC-4: two simulations can run side by side, each in half the terminal.

### EntropyFlash Lifecycle

`EntropyFlash` reads `SimState::entropy_events` each frame. For each `InjectionMarker` whose `generation` matches the current generation (new this frame), it creates an `ActiveFlash` with `frames_remaining = decay_frames`.

Each frame:
1. Decrement `frames_remaining` for all active flashes.
2. Remove flashes where `frames_remaining == 0`.
3. For each remaining flash, blend `self.color` into the FrameBuffer cell's `bg` at intensity `frames_remaining as f32 / decay_frames as f32`.

This satisfies AC-5: entropy injections produce a brief colored flash that decays over several frames.

### AnimationSpeed Frame Gating

AnimationSpeed decouples visual framerate from simulation step rate using a fractional accumulator:

- Each render tick, add `speed_multiplier` to `accumulated_steps`.
- If `accumulated_steps >= 1.0`, subtract 1.0 and return `should_render = true`.
- Otherwise, return `should_render = false` (skip this visual frame).

At `speed_multiplier = 0.5`, every other visual frame is rendered -- the simulation runs at full speed but the display updates at half rate. At `speed_multiplier = 2.0`, the display requests two simulation steps per visual frame (the simulation loop must cooperate by stepping twice before publishing).

For slow-motion (`speed_multiplier < 1.0`), the effect interpolates between the last-rendered and current FrameBuffer states using opacity blending, producing smooth slow-motion rather than choppy frame skipping.

This satisfies AC-6: animation speed 0.5x with 60 steps/sec produces 30 FPS visual output.

### Runtime Toggle Keybinds

All effects are individually toggleable via keybinds (NFR-2). The keybind mapping is:

| Key | Effect | Default |
|-----|--------|---------|
| `t` | SmoothTransition | enabled |
| `g` | GlowEffect | disabled |
| `z` | ZoomPan | disabled |
| `+` / `-` | Zoom in / out (when ZoomPan enabled) | -- |
| arrows | Pan (when ZoomPan enabled) | -- |
| `s` | SplitScreen cycle layout | disabled |
| `e` | EntropyFlash | enabled |
| `[` / `]` | AnimationSpeed slower / faster | -- |

These keybinds integrate with SPEC-010's runtime keybinding system (FR-7). The effect pipeline exposes `toggle(index)` and `list()` methods so the CLI can wire up keybinds without knowing effect internals.

## Data Model

```
FrameBuffer
  cells: Vec<FrameCell>       -- width * height, row-major
  width: usize
  height: usize

FrameCell
  age: u16                     -- simulation cell age
  fg: Color                    -- foreground (cell character color)
  bg: Color                    -- background (glow, flash overlays)
  opacity: f32                 -- for smooth transitions
  flash_intensity: f32         -- entropy flash overlay

EffectPipeline
  effects: Vec<Box<dyn Effect>>
  enabled: Vec<bool>           -- parallel toggle array

SimState
  cells: &[u16]               -- flat 2D-projected cell buffer
  width, height: usize
  generation: u64
  live_count: u64
  entropy_events: &[InjectionMarker]
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Stacked effects exceed 30 FPS budget (NFR-1) | Visual stuttering | Medium -- glow box filter is the most expensive | Profile each effect independently. GlowEffect uses a scratch buffer to avoid allocation. The box filter radius is capped at 4 cells. If still slow, switch to a separable two-pass box filter (O(width*height*radius) instead of O(width*height*radius^2)). |
| SplitScreen with multiple simulations doubles CPU cost | Frame drops with 4 panes | Low -- terminal size limits pane count to 4 | Each pane renders a smaller region. Total cell count stays the same as a single full-screen render. The simulation step cost is the real concern, not rendering. |
| ZoomPan nearest-neighbor sampling shows aliasing at high zoom | Blocky appearance at 4x+ zoom | Low -- terminal cells are already blocky | Acceptable for terminal rendering. At high zoom, individual cells are multiple terminal characters wide -- aliasing is not visible. |
| AnimationSpeed interpolation for slow-motion requires two frame snapshots | Extra memory for interpolation buffer | Very low -- one additional FrameBuffer-sized allocation (~100 KiB) | Acceptable. Only allocated when speed_multiplier < 1.0. |
| EntropyFlash accumulates unbounded active flashes during high-entropy injection | Memory growth, per-frame scan cost | Low -- decay_frames is small (default 5) | Cap active_flashes at `width * height` (one flash per cell max). New flashes at the same position replace existing ones. |

## ADR-001: FrameBuffer Intermediate over Direct Terminal Mutation

**Status:** Accepted

**Context:** Effects could mutate the ratatui buffer directly during `Renderer::render_frame`, or they could operate on an intermediate `FrameBuffer` that is converted to terminal output in a final pass.

**Decision:** Introduce `FrameBuffer` as an intermediate between `FrameState` and the `Renderer`.

**Consequences:**
- Effects are composable and testable without a terminal. Unit tests can assert on FrameBuffer contents.
- One extra buffer copy per frame (FrameBuffer to ratatui buffer). At 10,000 cells this is ~160 KiB -- negligible.
- The Renderer becomes simpler: it only converts FrameBuffer to terminal output. Effect logic is fully contained in the effect implementations.
- Effects that need to read neighboring cells (GlowEffect) can do so without worrying about ratatui buffer layout.

## ADR-002: Trait Object Pipeline over Static Effect Chain

**Status:** Accepted

**Context:** The effect chain could be a fixed sequence of concrete types (zero-cost, but closed) or a `Vec<Box<dyn Effect>>` (one vtable call per effect per frame, but open and toggleable at runtime).

**Decision:** Use `Vec<Box<dyn Effect>>` with a parallel `Vec<bool>` for enable/disable state.

**Consequences:**
- Effects can be toggled at runtime without recompilation or reconstruction.
- New effects can be added by implementing the `Effect` trait -- no changes to pipeline code.
- Per-frame cost is one vtable dispatch per enabled effect (6 effects at 60 FPS = 360 virtual calls/sec). The cost of each effect's `apply` body dominates by orders of magnitude.
- The parallel `Vec<bool>` avoids the complexity of removing/reinserting effects when toggling. Disabled effects are simply skipped.

## ADR-003: Box Filter over Gaussian Blur for Glow

**Status:** Accepted

**Context:** Glow could use a Gaussian blur kernel (smooth falloff, higher quality) or a box filter (uniform weight, simpler and faster).

**Decision:** Use a box filter for the glow density computation.

**Consequences:**
- Box filter is trivially implementable: count live cells in a rectangular region. No floating-point kernel weights.
- Visual quality difference is minimal at terminal resolution -- each "pixel" is a character cell, so the coarse grid masks the difference between Gaussian and box falloff.
- If the visual result is unsatisfactory, upgrading to a separable Gaussian (two-pass, still O(n*radius)) is straightforward without changing the Effect trait interface.
- Performance: at radius 2, the box is 5x5 = 25 cells. For a 200x50 grid, that is 250K comparisons per frame -- fast enough.
