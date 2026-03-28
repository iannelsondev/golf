# SPEC-009: Visual Effects -- Integration

## Consumed Interfaces

### SPEC-001 (Core Engine)

- Consumes: `NdGrid<N>`, `Cell` (u16 type alias), `SimulationEngine<N>`, `EntropyEvent<N>`
- Version: SPEC-001 as of main branch at implementation time
- Contract: `Cell` is `u16` where 0 = dead, >0 = age. `NdGrid<N>::cells()` returns `&[Cell]` as a flat row-major buffer. `NdGrid<N>::extents()` returns `[usize; N]`. `SimulationEngine<N>::generation()` returns `u64`.
- Usage: `SimState` wraps the projected cell buffer and generation counter from the simulation engine. Effects read cell ages to determine density, transition state, and coloring. `EntropyEvent::SetCells` coordinates are mapped to 2D for `InjectionMarker` creation.

### SPEC-002 (Rendering)

- Consumes: `FrameState`, `FrameSnapshot`, `Renderer`, `ColorPalette`, `ProjectionMode`, `RenderConfig`
- Version: SPEC-002 as of main branch at implementation time
- Contract: `FrameState::read()` returns `Arc<FrameSnapshot>`. `FrameSnapshot` contains `cells: Vec<Cell>`, `width: usize`, `height: usize`, `generation: u64`, `live_count: u64`. `ColorPalette::color_for_age(age: u16, max_age_hint: u16) -> Color`. `ProjectionMode` enum variants: `Flat2D`, `Slice`, `Flatten`, `Rotate`.
- Usage: The `FrameBuffer::load_from_snapshot` method reads a `FrameSnapshot` and applies a `ColorPalette` to produce initial cell colors before effects run. `SplitScreen` panes each reference a `ProjectionMode` to allow different projections in different panes. The `Renderer` is modified to accept a `FrameBuffer` as input (new method `render_from_buffer`) instead of only reading from `FrameState` directly.

#### Renderer Modification

SPEC-009 adds one method to `Renderer`:

```rust
impl Renderer {
    /// Render a pre-processed FrameBuffer to the terminal.
    /// This is called after EffectPipeline::apply_all() has run.
    pub fn render_from_buffer(&mut self, buf: &FrameBuffer) -> Result<()>;
}
```

The existing `render_frame(&mut self, state: &FrameState)` method remains for backward compatibility (it internally creates a FrameBuffer, applies the pipeline, and calls `render_from_buffer`). The render loop changes from:

```
FrameState -> Renderer::render_frame
```

to:

```
FrameState -> FrameBuffer::load_from_snapshot -> EffectPipeline::apply_all -> Renderer::render_from_buffer
```

### SPEC-003 (Entropy Trait)

- Consumes: `EntropyEvent<N>` variant information for injection visualization
- Version: SPEC-003 as of main branch at implementation time
- Contract: `EntropyEvent::SetCells` carries `coords: Vec<[usize; N]>` indicating where cells were injected. The coordinates are projected to 2D using the active `ProjectionMode` before being stored as `InjectionMarker` entries in `SimState`.
- Usage: `EntropyFlash` reads `SimState::entropy_events` to know where to place visual flash markers. The entropy source name is carried through for potential per-source color coding in future iterations.

## Provided Interfaces

### To SPEC-010 (CLI & Config)

- Provides: `EffectPipeline`, `Effect` trait, all concrete effect types, `FrameBuffer`
- Contract: `EffectPipeline::new()` creates an empty pipeline. `EffectPipeline::push(effect) -> usize` adds an effect and returns its toggle index. `EffectPipeline::toggle(index) -> bool` toggles an effect and returns the new enabled state. `EffectPipeline::list() -> Vec<(&str, bool)>` returns effect names and states for display.
- Usage: SPEC-010 constructs the `EffectPipeline` from CLI flags and TOML config. It wires runtime keybinds to `toggle()` calls. The `golf run` subcommand accepts `--effect` flags (e.g., `--effect glow --effect smooth`) to enable specific effects at startup. The TOML config supports an `[effects]` section:

```toml
[effects]
smooth_transition = true
smooth_blend_frames = 3
glow = false
glow_radius = 2
glow_threshold = 3
zoom_pan = false
split_screen = "single"  # "single", "h2", "v2", "quad"
entropy_flash = true
entropy_flash_decay = 5
animation_speed = 1.0
```

### To SPEC-011 (MCP Interface)

- Provides: `EffectPipeline::toggle(index)`, `EffectPipeline::list()`, `ZoomPan` control methods, `AnimationSpeed` control methods
- Contract: MCP tools can toggle effects, adjust zoom/pan, and change animation speed programmatically. The pipeline's `list()` method provides the current state for MCP tool responses.

## Integration Points

### Render Loop Integration

The primary integration point is the render loop in `golf-render`. The current loop (SPEC-002) is:

```
loop {
    let snapshot = frame_state.read();
    renderer.render_frame(&snapshot);
    // handle input
}
```

After SPEC-009, the loop becomes:

```
loop {
    let snapshot = frame_state.read();
    let sim_state = SimState::from_snapshot(&snapshot, &injection_markers);

    if animation_speed.should_render(snapshot.generation) {
        frame_buffer.load_from_snapshot(&snapshot, palette);
        effect_pipeline.apply_all(&mut frame_buffer, &sim_state);
        renderer.render_from_buffer(&frame_buffer)?;
    }

    // handle input (keybinds may toggle effects, adjust zoom, etc.)
}
```

### Injection Marker Collection

`InjectionMarker` entries are collected by the simulation coordinator (the component that bridges `SimulationEngine` and `Renderer`). When `SimulationEngine::inject_events()` is called, the coordinates from `SetCells` events are projected to 2D and stored in a ring buffer. The ring buffer is sized to `max_decay_frames * max_events_per_step` (default 5 * 100 = 500 entries). Older entries expire naturally when `EntropyFlash` checks their generation against `decay_frames`.

### Terminal Resize

On terminal resize:
1. `FrameBuffer::resize()` is called to match new dimensions.
2. `SplitScreen::relayout()` recalculates pane positions.
3. `ZoomPan` recalculates viewport bounds.
4. `GlowEffect` reallocates its density scratch buffer.
5. `SmoothTransition` clears its transition map (forces clean state).

The `Renderer::handle_resize()` method from SPEC-002 is extended to trigger these cascading resizes through the `EffectPipeline`.

## Dependency Graph

```
SPEC-001 (Core Engine)
    |
    v
SPEC-002 (Rendering) <--- SPEC-003 (Entropy Trait)
    |                         |
    v                         v
SPEC-009 (Visual Effects) <---+
    |
    v
SPEC-010 (CLI & Config)
SPEC-011 (MCP Interface)
```

SPEC-009 is a pure consumer of SPEC-001, SPEC-002, and SPEC-003. It adds no requirements to those specs -- it reads their published interfaces. The only modification to SPEC-002 is the addition of `render_from_buffer` on `Renderer`, which is additive (existing method remains).
