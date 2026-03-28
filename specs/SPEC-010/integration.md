# SPEC-010: CLI & Config -- Integration

## Consumed Interfaces

### SPEC-001 (Core Engine)
- **Version:** SPEC-001 as of current main branch at implementation time
- **Consumes:**
  - `SimulationEngine<const N: usize>` -- constructed with extents, rules, edge behaviors
  - `NdGrid<N>` -- read via `engine.grid()` for state access
  - `RuleSet` -- constructed from CLI `--rules` flag or config, passed to engine
  - `RuleSet::from_preset(Preset)` -- map named rule strings ("conway", "highlife") to RuleSet instances
  - `Preset` enum -- map CLI/config rule names to preset variants
  - `EdgeBehavior` enum -- map CLI `--edge` flag to per-dimension edge config
  - `EntropyEvent<N>` -- constructed by entropy sources, fed to `engine.inject_events()`
  - `Cell` type alias (`u16`) -- read from grid for rendering
- **Contract:** SimulationEngine must expose `step()`, `inject_events()`, `grid()`, `generation()`, `rules()`, `set_rules()`. NdGrid must expose `extents()` and `cells()`.

### SPEC-002 (Rendering)
- **Version:** SPEC-002 as of current main branch at implementation time
- **Consumes:**
  - `Renderer` -- constructed with `RenderConfig`, called each frame via `render_frame()`
  - `RenderConfig` -- built from resolved config (palette, cell_char, trail_length, projection, overlay)
  - `ColorPalette` trait + named implementations (`Inferno`, `Ocean`, `Matrix`, `Neon`, `Grayscale`)
  - `palette_by_name(name: &str) -> Option<Box<dyn ColorPalette>>` -- map CLI palette name to palette instance
  - `ProjectionMode` enum -- map CLI `--projection` and `--slice-depth` to projection config
  - `CellChar` enum -- map CLI `--cell-char` to rendering mode
  - `FrameState` -- shared between simulation thread and renderer thread
  - `FrameSnapshot` -- published by simulation thread after each step
  - `OverlayStats` -- populated from engine state for the stats overlay
- **Contract:** Renderer must expose `new(config)`, `render_frame(state)`, `handle_resize()`, `cleanup()`, `update_config()`. palette_by_name must return a palette for all five named palettes.

### SPEC-003 (Entropy Trait)
- **Version:** SPEC-003 as of current main branch at implementation time
- **Consumes:**
  - `SourceRegistry` -- constructed in App, sources registered and started during app init
  - `SourceRegistry::register(source) -> SourceId` -- register each CLI-requested source
  - `SourceRegistry::start(id)` -- start all registered sources before the main loop
  - `SourceRegistry::drain(budget) -> Vec<EntropyEvent>` -- called once per simulation step
  - `SourceRegistry::list() -> Vec<(SourceId, SourceMetadata)>` -- used by `golf list-sources`
  - `SourceMetadata` -- displayed in list-sources output (name, kind, status, events/sec)
  - `SourceStatus` enum -- displayed in list-sources to show available/unavailable/error
  - `EntropySource` trait -- individual source implementations constructed by source factories
- **Contract:** SourceRegistry must expose `register()`, `start()`, `stop()`, `drain()`, `list()`. Each source's `metadata()` must return current status including availability.

### SPEC-009 (Visual Effects)
- **Version:** SPEC-009 as of current main branch at implementation time
- **Consumes:**
  - Effect pipeline types (glow, smooth transitions, zoom/pan state) -- applied during rendering
  - Effect toggle functions -- mapped to keybinds (ToggleGlow, ZoomIn/Out, etc.)
  - Animation speed control -- mapped to IncreaseSpeed/DecreaseSpeed keybinds
- **Contract:** Effects must be individually toggleable at runtime. Effect state must be modifiable without restarting the simulation.
- **Note:** SPEC-009 is in draft status. If the effect pipeline interface changes, this integration must be updated. The CLI keybind map (ToggleGlow, ZoomIn/Out, speed controls) depends on SPEC-009 exposing these as runtime-modifiable operations.

## Provided Interfaces

### To SPEC-011 (MCP Interface)
- **Provides:**
  - `ServeArgs` struct -- parsed CLI arguments for `golf serve` subcommand
  - `GolfConfig` / `ResolvedConfig` -- MCP server uses the same config resolution pipeline
  - `App` orchestrator pattern -- SPEC-011 may construct its own variant or extend App with MCP channels
  - Config resolution pipeline (`load_default -> merge preset -> apply CLI -> resolve`) -- reusable for MCP server startup
- **Contract:** The `Serve` subcommand variant exists in the `Command` enum. `ServeArgs` contains transport, tui, bind, and token fields. The config resolution pipeline is callable from SPEC-011's serve handler.

## Integration Patterns

### App Construction Sequence

```
1. Parse CLI (clap derive)
2. Load base config: GolfConfig::load_default()
3. If --preset: load preset, merge on top
4. Apply CLI overrides
5. Resolve to ResolvedConfig
6. Match on dimensions -> monomorphize engine
7. Construct SimulationEngine<N> with extents, rules, edges
8. Construct SourceRegistry, register + start all requested sources
9. Construct Renderer with RenderConfig derived from ResolvedConfig
10. Spawn input event loop thread
11. Enter main loop: drain -> inject -> step -> publish -> render -> handle_input
```

### Thread Model

```
Main thread (tokio runtime):
  - App::run() select! loop
  - Simulation stepping (drain, inject, step, publish)
  - Key action handling

Input thread (std::thread):
  - crossterm event polling
  - KeyAction channel sender

Render thread (std::thread):
  - Renderer::render_frame() loop
  - Reads FrameState (lock-free Arc read)

Source background tasks (tokio tasks):
  - One per active EntropySource
  - Push EntropyEvents through per-source mpsc channels
```

### Error Handling at Boundaries

- Unknown source name: collect all unknowns, print available sources, exit with code 1
- Missing CUDA/audio/network runtime: log warning per source, continue with remaining sources
- Config file parse error: print the TOML error with file path and line number, exit with code 1
- Preset not found: print available presets, exit with code 1
- Terminal too small for requested grid: auto-size to terminal dimensions, log the adjustment
