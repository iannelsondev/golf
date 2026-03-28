# SPEC-010: CLI & Config -- Design

## Signatures

### Files Created or Modified

- `crates/golf-cli/src/lib.rs` -- crate root, re-exports
- `crates/golf-cli/src/cli.rs` -- `GolfCli` clap derive struct, subcommand enum
- `crates/golf-cli/src/config.rs` -- `GolfConfig` serde struct, TOML loading, CLI merge
- `crates/golf-cli/src/preset.rs` -- preset save/load/list operations
- `crates/golf-cli/src/demo.rs` -- `DemoRegistry`, named demo configs
- `crates/golf-cli/src/keybind.rs` -- `KeyAction` enum, crossterm event loop, key mapping
- `crates/golf-cli/src/app.rs` -- `App` orchestrator wiring engine + renderer + sources + effects
- `crates/golf-cli/src/main.rs` -- entrypoint, subcommand dispatch
- `crates/golf-cli/Cargo.toml` -- crate manifest (clap, serde, toml, dirs, crossterm, tokio)

### Types Introduced

```rust
/// Top-level CLI parsed by clap derive.
#[derive(Parser)]
#[command(name = "golf", about = "N-dimensional cellular automata framework")]
pub struct GolfCli {
    #[command(subcommand)]
    pub command: Command,
    /// Verbosity level (-v, -vv, -vvv)
    #[arg(short, long, action = ArgAction::Count, global = true)]
    pub verbose: u8,
}

/// Subcommands.
#[derive(Subcommand)]
pub enum Command {
    /// Start a simulation
    Run(RunArgs),
    /// List available entropy sources and their status
    ListSources,
    /// Run a curated demo
    Demo(DemoArgs),
    /// Manage presets
    Preset(PresetArgs),
    /// Start as an MCP server (see SPEC-011)
    Serve(ServeArgs),
}

/// Arguments for `golf run`.
#[derive(Args)]
pub struct RunArgs {
    /// Number of dimensions (default: 2)
    #[arg(short, long, default_value_t = 2)]
    pub dimensions: usize,
    /// Rule preset name or B/S notation (e.g., "conway", "B36/S23")
    #[arg(short, long, default_value = "conway")]
    pub rules: String,
    /// Entropy sources to activate (repeatable)
    #[arg(short, long)]
    pub source: Vec<String>,
    /// Color palette name
    #[arg(short, long)]
    pub palette: Option<String>,
    /// Grid width (default: terminal width)
    #[arg(long)]
    pub width: Option<usize>,
    /// Grid height (default: terminal height)
    #[arg(long)]
    pub height: Option<usize>,
    /// Edge behavior per dimension
    #[arg(long, default_value = "toroidal")]
    pub edge: String,
    /// Cell rendering mode
    #[arg(long)]
    pub cell_char: Option<String>,
    /// Trail length in frames (0 = disabled)
    #[arg(long)]
    pub trail_length: Option<u8>,
    /// N-to-2D projection mode
    #[arg(long)]
    pub projection: Option<String>,
    /// Slice depth for slice projection
    #[arg(long)]
    pub slice_depth: Option<usize>,
    /// Load a named preset (merged before CLI flags)
    #[arg(long)]
    pub preset: Option<String>,
    /// Disable TUI overlay
    #[arg(long)]
    pub no_overlay: bool,
}

/// Arguments for `golf demo`.
#[derive(Args)]
pub struct DemoArgs {
    /// Demo name (e.g., gpu-fire, audio-pulse, 3d-slice)
    pub name: Option<String>,
    /// List available demos
    #[arg(long)]
    pub list: bool,
}

/// Arguments for `golf preset`.
#[derive(Args)]
pub struct PresetArgs {
    #[command(subcommand)]
    pub action: PresetAction,
}

#[derive(Subcommand)]
pub enum PresetAction {
    /// Save current config as a named preset
    Save { name: String },
    /// Load and display a preset
    Show { name: String },
    /// List all saved presets
    List,
    /// Delete a preset
    Delete { name: String },
}

/// Arguments for `golf serve` (delegated to SPEC-011).
#[derive(Args)]
pub struct ServeArgs {
    /// Transport: "stdio" or "sse"
    #[arg(long, default_value = "stdio")]
    pub transport: String,
    /// Enable TUI alongside MCP server
    #[arg(long)]
    pub tui: bool,
    /// SSE bind address (only with --transport sse)
    #[arg(long, default_value = "127.0.0.1:3000")]
    pub bind: String,
    /// Bearer token for SSE transport authentication
    #[arg(long)]
    pub token: Option<String>,
}

/// Unified configuration deserialized from TOML.
/// Every field is Optional so partial configs merge cleanly.
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct GolfConfig {
    pub dimensions: Option<usize>,
    pub rules: Option<String>,
    pub sources: Option<Vec<String>>,
    pub palette: Option<String>,
    pub width: Option<usize>,
    pub height: Option<usize>,
    pub edge: Option<String>,
    pub cell_char: Option<String>,
    pub trail_length: Option<u8>,
    pub projection: Option<String>,
    pub slice_depth: Option<usize>,
    pub show_overlay: Option<bool>,
    pub target_fps: Option<u32>,
}

/// Actions triggered by runtime keybindings.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum KeyAction {
    Quit,
    PauseResume,
    StepOne,
    CyclePalette,
    ToggleOverlay,
    ZoomIn,
    ZoomOut,
    PanUp,
    PanDown,
    PanLeft,
    PanRight,
    NextProjection,
    ToggleGlow,
    IncreaseSpeed,
    DecreaseSpeed,
}

/// A named demo configuration.
pub struct DemoEntry {
    pub name: &'static str,
    pub description: &'static str,
    pub config: GolfConfig,
}

/// Registry of built-in demos.
pub struct DemoRegistry {
    demos: Vec<DemoEntry>,
}
```

### Functions / Contracts

```rust
// --- GolfConfig ---
impl GolfConfig {
    /// Load from the default config file (~/.config/golf/config.toml).
    /// Returns Default if the file does not exist.
    pub fn load_default() -> Result<Self>;

    /// Load from a specific TOML file path.
    pub fn load_from(path: &Path) -> Result<Self>;

    /// Merge another config on top of self. Non-None fields in `other` overwrite self.
    pub fn merge(&mut self, other: &GolfConfig);

    /// Apply CLI RunArgs overrides. CLI flags always win.
    pub fn apply_cli_overrides(&mut self, args: &RunArgs);

    /// Resolve a complete config with defaults applied for any remaining None fields.
    pub fn resolve(self) -> ResolvedConfig;

    /// Save to a TOML file.
    pub fn save_to(&self, path: &Path) -> Result<()>;
}

/// Fully resolved config with no Option fields. Produced by GolfConfig::resolve().
pub struct ResolvedConfig {
    pub dimensions: usize,
    pub rules: String,
    pub sources: Vec<String>,
    pub palette: String,
    pub width: usize,
    pub height: usize,
    pub edge: String,
    pub cell_char: String,
    pub trail_length: u8,
    pub projection: String,
    pub slice_depth: usize,
    pub show_overlay: bool,
    pub target_fps: u32,
}

// --- Config resolution order ---
// 1. GolfConfig::load_default()           (base: ~/.config/golf/config.toml)
// 2. .merge(preset_config)                (if --preset specified)
// 3. .apply_cli_overrides(run_args)       (CLI flags always win)
// 4. .resolve()                           (fill remaining Nones with hardcoded defaults)

// --- Preset operations ---
/// Directory: ~/.config/golf/presets/<name>.toml
pub fn preset_dir() -> PathBuf;
pub fn preset_save(name: &str, config: &GolfConfig) -> Result<()>;
pub fn preset_load(name: &str) -> Result<GolfConfig>;
pub fn preset_list() -> Result<Vec<String>>;
pub fn preset_delete(name: &str) -> Result<()>;

// --- DemoRegistry ---
impl DemoRegistry {
    /// Build the registry with all built-in demos.
    pub fn new() -> Self;

    /// Look up a demo by name.
    pub fn get(&self, name: &str) -> Option<&DemoEntry>;

    /// List all demos (name + description).
    pub fn list(&self) -> &[DemoEntry];
}

// --- KeyAction mapping ---
/// Map a crossterm KeyEvent to a KeyAction. Returns None for unmapped keys.
pub fn map_key(event: &KeyEvent) -> Option<KeyAction>;

/// Run the input event loop on a dedicated thread.
/// Reads crossterm events, maps to KeyAction, sends through channel.
/// Exits when it receives a Quit action or the sender is dropped.
pub fn run_input_loop(tx: mpsc::Sender<KeyAction>) -> JoinHandle<()>;

// --- App orchestrator ---
pub struct App {
    engine: SimulationEngine<N>,   // type-erased via dispatch enum, see Architecture
    registry: SourceRegistry,
    renderer: Renderer,
    config: ResolvedConfig,
    paused: bool,
}

impl App {
    /// Construct from resolved config. Wires engine, registry, renderer.
    pub fn new(config: ResolvedConfig) -> Result<Self>;

    /// Main loop: drain entropy, step engine, render frame, handle input.
    pub async fn run(&mut self) -> Result<()>;

    /// Handle a single key action.
    fn handle_action(&mut self, action: KeyAction);
}
```

## Architecture

### Config Resolution Pipeline

Configuration follows a strict layering with later layers overriding earlier ones:

```
Hardcoded defaults
    |
    v
~/.config/golf/config.toml  (GolfConfig::load_default)
    |
    v
Preset file                  (if --preset specified)
    |
    v
CLI flags                    (always win)
    |
    v
ResolvedConfig               (no Option fields, ready to use)
```

`GolfConfig` uses `Option<T>` for every field so that partial configs merge cleanly. The `merge` method only overwrites fields where the incoming config has `Some`. The `resolve` method fills any remaining `None` fields with hardcoded defaults:

| Field | Default |
|-------|---------|
| dimensions | 2 |
| rules | "conway" |
| sources | ["random"] |
| palette | "inferno" |
| width | terminal width |
| height | terminal height - 1 |
| edge | "toroidal" |
| cell_char | "full-block" |
| trail_length | 5 |
| projection | "flat2d" |
| slice_depth | 0 |
| show_overlay | true |
| target_fps | 60 |

### Dimension Dispatch

The `SimulationEngine<const N: usize>` requires a compile-time dimension. The CLI receives dimensions as a runtime integer. The app bridges this with a match-based dispatch:

```rust
match config.dimensions {
    2 => run_with_engine::<2>(config).await,
    3 => run_with_engine::<3>(config).await,
    4 => run_with_engine::<4>(config).await,
    n => {
        tracing::error!(dimensions = n, "unsupported dimension count (max 4)");
        return Err(anyhow!("dimensions must be 2, 3, or 4"));
    }
}
```

This monomorphizes for N=2, 3, and 4. Higher dimensions are rejected with a clear error. This matches the SPEC-001 design guidance that N>4 is rare and expensive.

### Source Stacking

Multiple `--source` flags are collected into a `Vec<String>`. During app construction:

1. For each source name, look up a factory function in a static map.
2. Construct the `EntropySource` implementation.
3. Register it with the `SourceRegistry`.
4. Start all registered sources.

If a source is unavailable (e.g., "gpu" without CUDA), log a warning and skip it. The simulation runs with whatever sources are available, falling back to "random" if all requested sources fail.

### Input Event Loop

A dedicated thread runs the crossterm event loop:

```
[crossterm raw mode events]
    |
    v
run_input_loop thread
    |  map_key(event) -> Option<KeyAction>
    v
mpsc::channel<KeyAction>
    |
    v
App::run() select! branch
    |
    v
App::handle_action(action)
```

The main `App::run` loop uses `tokio::select!` to multiplex:
1. A timer tick for simulation stepping (at target FPS).
2. The key action channel receiver.
3. (Future: MCP command channel from SPEC-011.)

Key actions are handled immediately. Pause/resume toggles a `paused` flag that skips the drain-step-render cycle. Step-one forces a single cycle regardless of pause state.

### Demo Registry

Demos are static `GolfConfig` instances with opinionated settings:

| Demo | Sources | Rules | Palette | Projection | Notes |
|------|---------|-------|---------|------------|-------|
| `gpu-fire` | gpu, random | conway | inferno | flat2d | VRAM noise with fire palette |
| `audio-pulse` | audio, random | highlife | neon | flat2d | Microphone FFT mapped to cells |
| `3d-slice` | random | 3d-life | ocean | slice(z=25) | 3D grid with slice viewer |
| `minimal` | random | conway | grayscale | flat2d | Clean, no effects |
| `chaos` | random, gpu, audio | day-night | neon | flat2d | Everything at once |

When a user runs `golf demo gpu-fire`, the demo's `GolfConfig` is loaded, then CLI overrides are applied on top (so `golf demo gpu-fire --palette ocean` works).

### Preset File Format

Presets are plain TOML files stored at `~/.config/golf/presets/<name>.toml`. They use the same schema as `GolfConfig`:

```toml
dimensions = 2
rules = "conway"
sources = ["random", "gpu"]
palette = "inferno"
trail_length = 8
show_overlay = false
```

`preset save` serializes the current `GolfConfig` (after resolution) to the named file. `preset load` deserializes and feeds it into the config resolution pipeline as the preset layer.

### Keybind Map

| Key | Action |
|-----|--------|
| `q` / `Esc` | Quit |
| `Space` | PauseResume |
| `n` | StepOne |
| `c` | CyclePalette |
| `o` | ToggleOverlay |
| `+` / `=` | ZoomIn |
| `-` | ZoomOut |
| `Up` | PanUp |
| `Down` | PanDown |
| `Left` | PanLeft |
| `Right` | PanRight |
| `p` | NextProjection |
| `g` | ToggleGlow |
| `]` | IncreaseSpeed |
| `[` | DecreaseSpeed |

The key map is a static lookup table. Custom keybinds are out of scope for v1.

## Data Model

```
GolfConfig
  dimensions: Option<usize>
  rules: Option<String>
  sources: Option<Vec<String>>
  palette: Option<String>
  ... (all Option<T> for merge semantics)

ResolvedConfig
  dimensions: usize
  rules: String
  sources: Vec<String>
  palette: String
  ... (all concrete, no Options)

App
  engine: SimulationEngine<N>  (dispatched via match on N)
  registry: SourceRegistry
  renderer: Renderer
  config: ResolvedConfig
  paused: bool
  key_rx: mpsc::Receiver<KeyAction>
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Config TOML deserialization fails silently on unknown fields | User typos ignored, confusing behavior | Medium | Use `serde(deny_unknown_fields)` on GolfConfig. Report the bad field name in the error message. |
| Dimension dispatch match does not cover a user's requested N | Runtime panic or confusing error | Low | Explicit match arms for 2, 3, 4 with a clear error for other values. Document the limit. |
| crossterm event loop blocks on stdin, preventing clean shutdown | App hangs on quit | Medium | Set a poll timeout on crossterm events (100ms). Check a shutdown flag between polls. |
| Preset directory does not exist on first run | File write fails | Medium | Create `~/.config/golf/presets/` on first preset save if it does not exist. |
| Source factory lookup fails for an unknown source name | User sees cryptic error | Medium | Collect all unknown source names and report them in a single error listing available sources. |

## ADR-001: Option-Based Config Merge over Builder Pattern

**Status:** Accepted

**Context:** Configuration comes from three layers (file, preset, CLI). The layers need to merge with "last writer wins" semantics. A builder pattern would require the caller to set fields in order. An `Option<T>`-per-field struct allows any layer to set any subset of fields.

**Decision:** Use `GolfConfig` with `Option<T>` for every field. Merge by overwriting `None` with `Some`.

**Consequences:**
- Any layer can be a partial config. A TOML file with only `palette = "ocean"` is valid.
- The merge function is trivial: `self.field = other.field.or(self.field)` for each field.
- The `resolve()` step guarantees all fields have values before the app starts.
- Slightly more boilerplate than a builder, but the merge semantics are explicit and testable.

## ADR-002: Static Dimension Dispatch over Dynamic Trait Object

**Status:** Accepted

**Context:** `SimulationEngine<const N: usize>` is a const generic. The CLI provides N at runtime. Two options: (a) match on N and monomorphize for each supported value, (b) erase the const generic behind a trait object.

**Decision:** Static dispatch via match on N for values 2, 3, 4.

**Consequences:**
- Each supported dimension compiles a separate monomorphization. Binary size increases by roughly 3x the engine code.
- No vtable overhead in the hot simulation loop.
- Adding N=5 or N=6 is a one-line match arm addition, but the binary cost grows linearly.
- Rejecting N>4 is acceptable -- SPEC-001 documents that N>4 is impractical due to 3^N neighbor explosion.
