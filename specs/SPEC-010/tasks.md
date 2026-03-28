# SPEC-010: CLI & Config -- Tasks

## Task 1: GolfConfig and TOML Loading
**AC coverage:** AC-5
**Files:** `crates/golf-cli/src/config.rs`

1. Define `GolfConfig` struct with `Option<T>` fields and serde derive (`Serialize`, `Deserialize`, `deny_unknown_fields`)
2. Implement `load_default()` -- resolve `~/.config/golf/config.toml` via `dirs::config_dir()`, return `Default` if missing
3. Implement `load_from(path)` -- read file, deserialize with `toml::from_str`, wrap errors with path context
4. Implement `merge(&mut self, other)` -- overwrite self fields where other is `Some`
5. Implement `resolve(self) -> ResolvedConfig` -- fill `None` fields with hardcoded defaults (dimensions=2, rules="conway", palette="inferno", etc.)
6. Implement `save_to(path)` -- serialize to TOML, write file
7. Write tests: merge semantics (later layer wins), resolve defaults, round-trip serialize/deserialize, deny_unknown_fields error

## Task 2: Clap CLI Parsing
**AC coverage:** AC-1, AC-2
**Files:** `crates/golf-cli/src/cli.rs`

1. Define `GolfCli` with `#[derive(Parser)]` and global `--verbose` flag
2. Define `Command` enum with `Run`, `ListSources`, `Demo`, `Preset`, `Serve` variants
3. Define `RunArgs` with all flags: `--dimensions`, `--rules`, `--source` (repeatable), `--palette`, `--width`, `--height`, `--edge`, `--cell-char`, `--trail-length`, `--projection`, `--slice-depth`, `--preset`, `--no-overlay`
4. Define `DemoArgs` with positional `name` and `--list` flag
5. Define `PresetArgs` with `PresetAction` subcommand enum (Save, Show, List, Delete)
6. Define `ServeArgs` with `--transport`, `--tui`, `--bind`, `--token`
7. Implement `apply_cli_overrides(&mut GolfConfig, &RunArgs)` -- map each CLI flag to the corresponding config field
8. Write tests: parse known good command lines, verify defaults, verify --source stacking produces Vec

## Task 3: Preset System
**AC coverage:** AC-6
**Files:** `crates/golf-cli/src/preset.rs`

1. Implement `preset_dir()` -- `dirs::config_dir() / "golf" / "presets"`
2. Implement `preset_save(name, config)` -- create dir if needed, serialize config to `<name>.toml`
3. Implement `preset_load(name)` -- read `<name>.toml`, deserialize to `GolfConfig`
4. Implement `preset_list()` -- glob `*.toml` in preset dir, strip extensions, return sorted names
5. Implement `preset_delete(name)` -- remove file, error if not found
6. Write tests: save then load round-trip, list returns saved names, delete removes file, load nonexistent returns error

## Task 4: Demo Registry
**AC coverage:** AC-4
**Files:** `crates/golf-cli/src/demo.rs`

1. Define `DemoEntry` struct (name, description, config)
2. Define `DemoRegistry` with a `Vec<DemoEntry>`
3. Implement `DemoRegistry::new()` -- populate with built-in demos:
   - `gpu-fire`: sources=["gpu","random"], rules="conway", palette="inferno"
   - `audio-pulse`: sources=["audio","random"], rules="highlife", palette="neon"
   - `3d-slice`: dimensions=3, sources=["random"], rules="3d-life", palette="ocean", projection="slice", slice_depth=25
   - `minimal`: sources=["random"], rules="conway", palette="grayscale"
   - `chaos`: sources=["random","gpu","audio"], rules="day-night", palette="neon"
4. Implement `get(name)` and `list()`
5. Write tests: get known demo returns correct config, get unknown returns None, list returns all demos

## Task 5: Keybind Handler
**AC coverage:** AC-7
**Files:** `crates/golf-cli/src/keybind.rs`

1. Define `KeyAction` enum with all variants (Quit, PauseResume, StepOne, CyclePalette, ToggleOverlay, ZoomIn/Out, Pan*, NextProjection, ToggleGlow, IncreaseSpeed, DecreaseSpeed)
2. Implement `map_key(KeyEvent) -> Option<KeyAction>` -- static match on key code and modifiers
3. Implement `run_input_loop(tx: mpsc::Sender<KeyAction>) -> JoinHandle<()>`:
   - Loop: `crossterm::event::poll(Duration::from_millis(100))`
   - On poll success: `crossterm::event::read()`, filter for `Event::Key`, map to action, send
   - On Quit action: send it and return
   - On channel closed (send error): return
4. Write tests: map_key for each documented keybind, map_key returns None for unmapped keys

## Task 6: App Orchestrator and Main Loop
**AC coverage:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-7
**Files:** `crates/golf-cli/src/app.rs`, `crates/golf-cli/src/main.rs`

1. Implement dimension dispatch function:
   ```rust
   async fn dispatch_run(config: ResolvedConfig) -> Result<()> {
       match config.dimensions {
           2 => run_with_engine::<2>(config).await,
           3 => run_with_engine::<3>(config).await,
           4 => run_with_engine::<4>(config).await,
           n => bail!("unsupported dimensions: {n} (max 4)"),
       }
   }
   ```
2. Implement `run_with_engine<const N: usize>(config)`:
   - Parse rules string to `RuleSet` (preset name or B/S notation)
   - Parse edge string to `[EdgeBehavior; N]`
   - Construct `SimulationEngine<N>`
   - Construct `SourceRegistry`, register and start sources
   - Construct `Renderer` with `RenderConfig`
   - Construct `FrameState`
   - Spawn input loop thread
   - Enter main loop with `tokio::select!`
3. Implement main loop:
   - Timer tick at target FPS interval
   - On tick (if not paused): drain registry, inject events, step engine, publish frame, render
   - On key action: handle_action (pause, step, palette cycle, overlay toggle, quit, zoom, pan)
   - On quit: cleanup renderer, stop all sources, exit
4. Implement `main()`:
   - Parse `GolfCli`
   - Initialize tracing subscriber (level from --verbose)
   - Match on Command:
     - `Run`: load config, apply preset, apply CLI, resolve, dispatch_run
     - `ListSources`: construct registry, probe all known source types, print table
     - `Demo`: look up demo, merge with CLI overrides, dispatch_run
     - `Preset`: delegate to preset functions
     - `Serve`: delegate to SPEC-011 serve handler
5. Write integration tests:
   - Config resolution: file + preset + CLI produces correct ResolvedConfig
   - ListSources: outputs table format with source names and status
   - Demo lookup: known demo name resolves, unknown demo errors

## Task 7: List Sources Command
**AC coverage:** AC-3
**Files:** `crates/golf-cli/src/app.rs` (or `crates/golf-cli/src/list.rs`)

1. Define source factory registry: a static map of source name to availability check function
2. For each known source type (random, gpu, audio, network, file), probe availability:
   - `random`: always available
   - `gpu`: check CUDA runtime availability (attempt `cudarc` init, catch error)
   - `audio`: check `cpal` default input device
   - `network`: check capture permissions
   - `file`: always available (requires a path argument at runtime)
3. Print a formatted table:
   ```
   Source     Status        Notes
   random     available
   gpu        unavailable   CUDA runtime not found
   audio      available     device: Built-in Microphone
   network    unavailable   requires CAP_NET_RAW
   file       available     pass --file <path> to use
   ```
4. Write tests: mock source factories to test table formatting for mixed available/unavailable
