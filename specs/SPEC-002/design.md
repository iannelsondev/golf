# SPEC-002: Rendering — Design

## Signatures

### Files Created or Modified

- `crates/golf-render/src/lib.rs` — crate root, public re-exports
- `crates/golf-render/src/renderer.rs` — `Renderer` struct, main render loop, differential redraw
- `crates/golf-render/src/palette.rs` — `ColorPalette` trait and named implementations
- `crates/golf-render/src/projection.rs` — `ProjectionMode` enum, N-to-2D projection logic
- `crates/golf-render/src/config.rs` — `RenderConfig`, `CellChar` enum
- `crates/golf-render/src/frame.rs` — `FrameState` double-buffer for sim-renderer decoupling
- `crates/golf-render/src/braille.rs` — braille sub-grid mapping (2x4 cells to U+2800-U+28FF)
- `crates/golf-render/src/trail.rs` — death trail ring buffer, fade alpha computation
- `crates/golf-render/src/overlay.rs` — stats overlay (FPS, generation, live cells, dimensions, rules)
- `crates/golf-render/Cargo.toml` — crate manifest (ratatui, crossterm)

### Types Introduced

```rust
/// How to render each simulation cell in the terminal.
pub enum CellChar {
    FullBlock,       // U+2588
    HalfBlock,       // U+2584 / U+2580
    Braille,         // 2x4 sub-grid mapped to U+2800-U+28FF
    Custom(char),
}

/// N-to-2D projection mode for dimensions > 2.
pub enum ProjectionMode {
    /// Direct 2D render (only first two dimensions).
    Flat2D,
    /// Show a 2D cross-section at a fixed depth along an axis.
    Slice { axis: usize, depth: usize },
    /// Overlay multiple layers with alpha blending (max-age-wins).
    Flatten,
    /// Rotate through a dimensional axis at a given angle (radians).
    Rotate { axis: usize, angle: f64 },
}

/// Rendering configuration, fully hot-reloadable.
pub struct RenderConfig {
    pub palette: Box<dyn ColorPalette>,
    pub cell_char: CellChar,
    pub trail_length: u8,         // frames a dead cell fades over (0 = instant disappear)
    pub show_overlay: bool,
    pub projection: ProjectionMode,
}

/// Per-cell death trail entry.
struct TrailEntry {
    death_generation: u64,
}

/// Ring buffer tracking recent deaths for a cell position.
struct TrailBuffer {
    entries: Vec<Option<TrailEntry>>,  // indexed by flat 2D position
    capacity: u8,                      // == RenderConfig.trail_length
}

/// Double-buffered frame state for sim-renderer decoupling.
/// The simulation thread writes to the back slot; the renderer reads the front slot.
pub struct FrameState {
    front: Arc<FrameSnapshot>,
    back: Mutex<FrameSnapshot>,
}

/// Snapshot of one frame's worth of data for the renderer.
pub struct FrameSnapshot {
    pub cells: Vec<Cell>,            // flat buffer, 2D-projected
    pub width: usize,
    pub height: usize,
    pub generation: u64,
    pub live_count: u64,
    pub dimensions: usize,           // N from the simulation
    pub rule_description: String,    // e.g., "B3/S23 Moore"
}

/// Simulation stats displayed in the overlay.
pub struct OverlayStats {
    pub fps: f64,
    pub generation: u64,
    pub live_cells: u64,
    pub dimensions: usize,
    pub rule_description: String,
    pub active_sources: Vec<String>,
}

/// Top-level renderer owning the terminal and render state.
pub struct Renderer {
    terminal: Terminal<CrosstermBackend<Stdout>>,
    config: RenderConfig,
    trail: TrailBuffer,
    prev_frame: Vec<Cell>,          // last rendered cells for differential redraw
    frame_times: VecDeque<Instant>, // rolling window for FPS calculation
}
```

### Functions / Traits / Contracts

```rust
// --- ColorPalette trait ---

/// Maps a cell age (0 = dead, 1+ = alive) to an RGB color.
/// Implementations define the gradient curve.
pub trait ColorPalette: Send + Sync {
    /// Color for a live cell at the given age. Age is always >= 1.
    fn color_for_age(&self, age: u16, max_age_hint: u16) -> Color;

    /// Color for a dead cell in its trail phase.
    /// `fade` ranges from 1.0 (just died) to 0.0 (fully faded).
    fn trail_color(&self, fade: f64) -> Color;

    /// Display name of this palette.
    fn name(&self) -> &str;
}

// Named implementations (all implement ColorPalette):
pub struct Inferno;    // black -> red -> orange -> yellow -> white
pub struct Ocean;      // dark blue -> cyan -> white
pub struct Matrix;     // dark green -> bright green -> white
pub struct Neon;       // deep purple -> magenta -> pink -> white
pub struct Grayscale;  // dark gray -> white

/// Look up a palette by name. Returns None for unknown names.
pub fn palette_by_name(name: &str) -> Option<Box<dyn ColorPalette>>;

// --- Projection ---

/// Project an N-dimensional grid to a 2D cell buffer.
/// Returns a flat Vec<Cell> of size width*height.
pub fn project_to_2d<const N: usize>(
    grid: &NdGrid<N>,
    mode: &ProjectionMode,
    width: usize,
    height: usize,
) -> Vec<Cell>;

// --- FrameState ---

impl FrameState {
    /// Create a new frame state for the given 2D dimensions.
    pub fn new(width: usize, height: usize) -> Self;

    /// Called by the simulation thread: write a new snapshot into the back buffer,
    /// then atomically swap front and back.
    pub fn publish(&self, snapshot: FrameSnapshot);

    /// Called by the render thread: clone the current front snapshot.
    /// Non-blocking (reads an Arc, no lock contention with publish).
    pub fn read(&self) -> Arc<FrameSnapshot>;
}

// --- Braille ---

/// Map a 2x4 sub-grid of Cell values to a single braille character (U+2800-U+28FF).
/// Each of the 8 dots corresponds to one cell: alive = dot raised, dead = dot lowered.
pub fn cells_to_braille(subgrid: [[Cell; 4]; 2]) -> char;

/// Render an entire 2D cell buffer in braille mode.
/// Output width = ceil(input_width / 2), output height = ceil(input_height / 4).
pub fn render_braille_buffer(
    cells: &[Cell],
    width: usize,
    height: usize,
) -> Vec<(char, Color)>;

// --- TrailBuffer ---

impl TrailBuffer {
    /// Create a trail buffer for the given 2D dimensions and trail length.
    pub fn new(width: usize, height: usize, trail_length: u8) -> Self;

    /// Record deaths: compare current frame to previous frame,
    /// mark newly-dead positions with the current generation.
    pub fn record_deaths(
        &mut self,
        prev: &[Cell],
        current: &[Cell],
        generation: u64,
    );

    /// Compute fade factor for a position. Returns 0.0 if no active trail,
    /// otherwise a value in (0.0, 1.0] based on age relative to trail_length.
    pub fn fade_at(&self, flat_index: usize, current_generation: u64) -> f64;
}

// --- Renderer ---

impl Renderer {
    /// Initialize the terminal (enter alternate screen, enable raw mode).
    pub fn new(config: RenderConfig) -> Result<Self>;

    /// Render one frame from the given FrameState.
    /// Performs differential redraw: only cells that changed since the last
    /// frame are written to the terminal buffer.
    pub fn render_frame(&mut self, state: &FrameState) -> Result<()>;

    /// Render the stats overlay in the top-right corner.
    fn render_overlay(&self, frame: &mut Frame, stats: &OverlayStats);

    /// Handle terminal resize: update internal dimensions, force full redraw.
    pub fn handle_resize(&mut self, width: u16, height: u16);

    /// Restore terminal state (leave alternate screen, disable raw mode).
    pub fn cleanup(&mut self) -> Result<()>;

    /// Update the render configuration at runtime (palette swap, toggle overlay, etc.).
    pub fn update_config(&mut self, config: RenderConfig);

    /// Current measured FPS based on rolling frame time window.
    pub fn current_fps(&self) -> f64;
}
```

## Architecture

### Simulation-Renderer Decoupling via FrameState

The simulation and renderer run on separate threads. They communicate through `FrameState`, which holds two `FrameSnapshot` values behind an `Arc`/`Mutex` arrangement:

1. The simulation thread calls `FrameState::publish(snapshot)` after each step. This acquires the back-buffer mutex, writes the snapshot, and atomically swaps the front `Arc`.
2. The renderer calls `FrameState::read()` which clones the current front `Arc`. This is lock-free on the read path -- the renderer never contends with the simulation.
3. If the simulation produces frames faster than the renderer consumes them, intermediate frames are silently dropped. The renderer always sees the latest state. This is intentional -- visual smoothness matters more than rendering every generation.

This satisfies NFR-2 (rendering never blocks simulation).

### Differential Redraw

The renderer maintains a `prev_frame: Vec<Cell>` buffer. On each frame:

1. Read the new snapshot from `FrameState`.
2. Compare each cell position against `prev_frame`.
3. Only write changed cells to the ratatui buffer.
4. After ratatui flushes, copy the new cells into `prev_frame`.

For typical Game of Life patterns where less than 20% of cells change per frame, this reduces terminal write volume by 80%+. Terminal I/O is the bottleneck for 60 FPS, so minimizing writes is critical.

On terminal resize, `prev_frame` is cleared to force a full redraw.

### Braille Rendering

In braille mode, each terminal character cell represents a 2x4 sub-grid of simulation cells. The Unicode braille range (U+2800 to U+28FF) encodes 8 dots as 8 bits:

```
Dot positions:     Bit positions:
[1] [4]            [0] [3]
[2] [5]            [1] [4]
[3] [6]            [2] [5]
[7] [8]            [6] [7]
```

A live cell (age > 0) raises the corresponding dot. The braille character is `0x2800 + bitmask`. This gives 4x density compared to full-block rendering: a 200x50 terminal can display an 400x200 simulation grid.

Color for each braille character is determined by the maximum age of any live cell in its 2x4 sub-grid. This preserves the palette gradient while keeping one color per terminal cell.

### Death Trail Effect

The trail system uses a flat `Vec<Option<TrailEntry>>` indexed by 2D position. When a cell transitions from alive to dead between frames:

1. `TrailBuffer::record_deaths` detects the transition by comparing prev and current cell buffers.
2. It stores a `TrailEntry { death_generation }` at that position.
3. During rendering, `fade_at(pos, current_gen)` computes: `fade = 1.0 - (current_gen - death_generation) as f64 / trail_length as f64`.
4. If `fade > 0.0`, the palette's `trail_color(fade)` is used instead of the background color.
5. When `fade <= 0.0`, the entry is considered expired and the cell renders as background.

Trail length is configurable (default 5 frames per AC-2). Setting `trail_length = 0` disables trails entirely.

### N-to-2D Projection

The `project_to_2d` function handles four modes:

- **Flat2D:** Direct copy of the first two dimensions. For N=2, this is identity. For N>2, uses the slice at index 0 of all higher dimensions.
- **Slice { axis, depth }:** Fixes one dimension at a constant depth and takes the 2D plane formed by the remaining two lowest dimensions. For a 3D grid with `axis=2, depth=5`, this shows the XY plane at Z=5.
- **Flatten:** Iterates over all positions in the higher dimensions. For each 2D position, takes the maximum cell age across all higher-dimensional layers. This produces a "heat overlay" showing the most active areas across all layers.
- **Rotate { axis, angle }:** Computes a rotation matrix in the plane defined by `axis` and the next dimension. Maps each 2D output pixel back through the inverse rotation to sample the grid. Uses nearest-neighbor sampling. Primarily useful for animated sweeps through 3D+ spaces.

### Color Palette Design

All palettes implement the same `ColorPalette` trait. The `color_for_age` method receives the cell's current age and a hint for the maximum age in the current frame (used for normalization). Palettes map `age / max_age_hint` to a position on their gradient curve.

Each palette defines a gradient as 3-5 control points. Interpolation between control points is linear in RGB space (sufficient for terminal colors where the palette is quantized to 256 colors or 24-bit true color depending on terminal capability).

### Stats Overlay

When `show_overlay` is true, the renderer draws a transparent overlay in the top-right corner of the terminal:

```
 FPS: 62.3 | Gen: 1,042
 Live: 3,217 / 10,000
 Dim: 2D | B3/S23 Moore
```

The overlay is rendered as a ratatui `Block` with a semi-transparent background (if the terminal supports it) or a solid dark background otherwise. It is always drawn last, on top of the cell grid.

FPS is computed from a rolling window of the last 60 frame timestamps. The window is a `VecDeque<Instant>` that is pruned each frame.

## Data Model

```
FrameState
  front: Arc<FrameSnapshot>    -- latest published frame (renderer reads this)
  back: Mutex<FrameSnapshot>   -- simulation writes here, then swaps

FrameSnapshot
  cells: Vec<Cell>              -- 2D-projected cell buffer (flat, row-major)
  width: usize
  height: usize
  generation: u64
  live_count: u64
  dimensions: usize
  rule_description: String

TrailBuffer
  entries: Vec<Option<TrailEntry>>  -- indexed by flat 2D position
  capacity: u8                      -- max trail length in frames

Renderer
  terminal: Terminal<CrosstermBackend<Stdout>>
  config: RenderConfig
  trail: TrailBuffer
  prev_frame: Vec<Cell>
  frame_times: VecDeque<Instant>
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Terminal does not support 24-bit true color | Palettes look wrong or unreadable | Medium -- many terminals support it, some do not | Detect color capability at startup via `COLORTERM` env var. Fall back to 256-color approximation. |
| Braille characters render incorrectly in some fonts | Density mode looks garbled | Low -- most monospace fonts include braille block | Detect at startup with a test write. Warn user and fall back to full-block mode. |
| `Arc` swap in FrameState causes a torn read if snapshot clone is non-atomic | Renderer sees inconsistent frame | Very low -- `Arc::clone` is atomic | The front pointer is only swapped via `Arc::swap` or equivalent atomic operation. `FrameSnapshot` is immutable once published. No torn reads possible. |
| Trail buffer memory for large grids (e.g., 400x200 braille) | 80K entries, each 9 bytes = ~720 KiB | Very low | Acceptable for terminal-sized grids. Cap at terminal size. |
| Projection rotation with nearest-neighbor sampling produces aliasing | Visual artifacts during rotation animation | Medium | Acceptable for terminal rendering where each "pixel" is a character cell. Bilinear interpolation is not worth the cost at this resolution. |

## ADR-001: Arc-Swap FrameState over Channel-Based Frame Passing

**Status:** Accepted

**Context:** The simulation and renderer need to exchange frame data without blocking each other. Two patterns were considered: (1) an `mpsc` channel where the sim sends frames and the renderer drains the latest, or (2) a shared `FrameState` with atomic swap.

**Decision:** Use `FrameState` with `Arc` swap.

**Consequences:**
- The renderer always reads the latest frame with no queue buildup. No need to drain stale frames from a channel.
- The simulation never blocks on a full channel -- it overwrites the back buffer and swaps.
- Slightly more complex than a channel, but avoids the "slow consumer" problem entirely.
- If we later need frame history (e.g., for recording), a channel can be added alongside FrameState.

## ADR-002: Trait Object for ColorPalette over Enum Dispatch

**Status:** Accepted

**Context:** Palettes could be an enum with match-based dispatch or a trait object (`Box<dyn ColorPalette>`). An enum is zero-cost but closed -- adding palettes requires modifying the enum. A trait object has vtable overhead but is open for extension.

**Decision:** Use `trait ColorPalette` with `Box<dyn ColorPalette>` in `RenderConfig`.

**Consequences:**
- Users and future specs (SPEC-009 Visual Effects) can define custom palettes without modifying golf-render.
- One vtable call per cell per frame. At 10,000 cells and 60 FPS, that is 600K virtual calls/sec -- negligible on modern hardware.
- The trait must be `Send + Sync` to cross the thread boundary in FrameState.

## ADR-003: Per-Cell Trail Entry over Bitmap Trail

**Status:** Accepted

**Context:** Death trails could be tracked with (1) a per-cell `Option<TrailEntry>` storing the death generation, or (2) a series of bitmaps (one per trail frame) where each bit indicates a death at that age.

**Decision:** Per-cell `Option<TrailEntry>` with generation timestamp.

**Consequences:**
- Simple fade computation: `(current_gen - death_gen) / trail_length`.
- Memory: one `Option<TrailEntry>` (9 bytes with padding) per 2D cell. For 200x50 = 10,000 cells, that is 90 KiB.
- No need to shift bitmaps each frame. The generation counter is monotonically increasing, so old entries naturally expire.
- If multiple deaths happen at the same position within the trail window, only the most recent is tracked. This is visually correct -- the most recent death dominates the fade.
