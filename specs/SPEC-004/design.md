# SPEC-004: Classic Sources -- Design

## Signatures

### Files Created or Modified

- `crates/golf-entropy/src/sources/mod.rs` -- module declarations for classic sources
- `crates/golf-entropy/src/sources/random.rs` -- `RandomSource` struct implementing `EntropySource`
- `crates/golf-entropy/src/sources/rle.rs` -- `RleParser` for Run Length Encoded pattern files
- `crates/golf-entropy/src/sources/life106.rs` -- `Life106Parser` for Life 1.06 plaintext format
- `crates/golf-entropy/src/sources/glider_gun.rs` -- `GliderGunSource` periodic pattern injector
- `crates/golf-entropy/src/sources/noise.rs` -- `NoiseSource` Perlin/simplex noise-based source
- `crates/golf-entropy/src/sources/pattern_library.rs` -- `PatternLibrary` bundled named patterns
- `crates/golf-formats/src/lib.rs` -- crate root, re-exports parsers (optional: parsers may live in golf-entropy directly; see ADR-001)
- `crates/golf-entropy/Cargo.toml` -- updated with `noise`, `rand` dependencies

### Types Introduced

```rust
/// Configuration for the random entropy source.
pub struct RandomConfig {
    pub density: f64,           // 0.0..=1.0, fraction of cells set alive
    pub reseed_interval: Option<Duration>,  // None = one-shot, Some = periodic re-injection
    pub dimensions: Vec<usize>, // grid extents to fill (e.g., [200, 50])
}

/// Random entropy source: fills grid cells based on configurable density.
/// On start, generates initial SetCells event. If reseed_interval is set,
/// periodically generates new SetCells events at that interval.
pub struct RandomSource {
    config: RandomConfig,
    tx: Option<mpsc::Sender<EntropyEvent>>,
    task: Option<JoinHandle<()>>,
}

/// Parsed pattern: a list of alive cell coordinates with optional metadata.
pub struct ParsedPattern {
    pub name: Option<String>,
    pub author: Option<String>,
    pub comments: Vec<String>,
    pub rule: Option<String>,         // e.g., "B3/S23"
    pub alive_cells: Vec<Vec<usize>>, // each inner Vec is an N-dimensional coordinate
    pub bounding_box: Vec<usize>,     // extent per dimension
}

/// RLE file parser.
pub struct RleParser;

/// Life 1.06 plaintext file parser.
pub struct Life106Parser;

/// Gosper glider gun injection source.
pub struct GliderGunConfig {
    pub interval: Duration,       // time between injections
    pub dimensions: Vec<usize>,   // grid extents (for random placement bounds)
}

pub struct GliderGunSource {
    config: GliderGunConfig,
    tx: Option<mpsc::Sender<EntropyEvent>>,
    task: Option<JoinHandle<()>>,
}

/// Perlin/simplex noise entropy source.
pub struct NoiseConfig {
    pub noise_type: NoiseType,
    pub scale: f64,             // spatial scale (lower = smoother)
    pub threshold: f64,         // noise value above which a cell is alive (0.0..=1.0)
    pub scroll_speed: f64,      // units per second through noise space
    pub tick_interval: Duration, // how often to re-evaluate and emit events
    pub dimensions: Vec<usize>,  // grid extents
}

pub enum NoiseType {
    Perlin,
    Simplex,
}

pub struct NoiseSource {
    config: NoiseConfig,
    tx: Option<mpsc::Sender<EntropyEvent>>,
    task: Option<JoinHandle<()>>,
}

/// Library of bundled classic patterns, loadable by name.
pub struct PatternLibrary {
    patterns: HashMap<String, Vec<Vec<usize>>>,
}
```

### Functions / Traits / Contracts

```rust
// --- RandomSource ---
impl RandomSource {
    pub fn new(config: RandomConfig) -> Self;
}

impl EntropySource for RandomSource {
    fn name(&self) -> &str;       // "random"
    fn kind(&self) -> &str;       // "classic"
    async fn start(&mut self) -> Result<Receiver<EntropyEvent>>;
    async fn stop(&mut self) -> Result<()>;
    fn metadata(&self) -> SourceMetadata;
}

// --- RleParser ---
impl RleParser {
    /// Parse an RLE-formatted string into a ParsedPattern.
    /// Handles header lines (x, y, rule), comment lines (#C, #N, #O),
    /// and the run-length encoded body (digits + b/o/$).
    pub fn parse(input: &str) -> Result<ParsedPattern>;

    /// Parse from a file path.
    pub fn parse_file(path: &Path) -> Result<ParsedPattern>;
}

// --- Life106Parser ---
impl Life106Parser {
    /// Parse a Life 1.06 formatted string into a ParsedPattern.
    /// Format: header line "#Life 1.06", then one "x y" coordinate pair per line.
    pub fn parse(input: &str) -> Result<ParsedPattern>;

    /// Parse from a file path.
    pub fn parse_file(path: &Path) -> Result<ParsedPattern>;
}

// --- GliderGunSource ---
impl GliderGunSource {
    pub fn new(config: GliderGunConfig) -> Self;
}

impl EntropySource for GliderGunSource {
    fn name(&self) -> &str;       // "glider-gun"
    fn kind(&self) -> &str;       // "classic"
    async fn start(&mut self) -> Result<Receiver<EntropyEvent>>;
    async fn stop(&mut self) -> Result<()>;
    fn metadata(&self) -> SourceMetadata;
}

// --- NoiseSource ---
impl NoiseSource {
    pub fn new(config: NoiseConfig) -> Self;
}

impl EntropySource for NoiseSource {
    fn name(&self) -> &str;       // "noise"
    fn kind(&self) -> &str;       // "classic"
    async fn start(&mut self) -> Result<Receiver<EntropyEvent>>;
    async fn stop(&mut self) -> Result<()>;
    fn metadata(&self) -> SourceMetadata;
}

// --- PatternLibrary ---
impl PatternLibrary {
    /// Construct the library with all bundled patterns.
    pub fn new() -> Self;

    /// Look up a pattern by name (case-insensitive).
    pub fn get(&self, name: &str) -> Option<&Vec<Vec<usize>>>;

    /// List all available pattern names.
    pub fn list(&self) -> Vec<&str>;

    /// Convert a named pattern into a ParsedPattern centered at the given offset.
    pub fn to_pattern(&self, name: &str, offset: Vec<usize>) -> Option<ParsedPattern>;

    /// Convert a named pattern into an ApplyPattern entropy event.
    pub fn to_event(&self, name: &str, offset: Vec<usize>) -> Option<EntropyEvent>;
}
```

## Architecture

### Source Lifecycle

All four `EntropySource` implementations (`RandomSource`, `GliderGunSource`, `NoiseSource`, and file-loaded patterns wrapped in a one-shot source) follow the same pattern:

1. `start()` creates a bounded `mpsc` channel, spawns a tokio task, stores the `Sender` and `JoinHandle`, and returns the `Receiver`.
2. The background task produces `EntropyEvent` values and sends them through the channel.
3. `stop()` drops the `Sender` (causing the task's send to fail) and awaits the `JoinHandle`.
4. Sources that are one-shot (random init, file load) send their events immediately on start and then exit the task. Sources that are periodic (reseed, glider gun, noise) loop on a `tokio::time::interval`.

### RandomSource Behavior

On `start()`, the background task:

1. Iterates all coordinates within `config.dimensions`.
2. For each coordinate, generates a random `f64` in `[0.0, 1.0)`.
3. If the value is less than `config.density`, includes the coordinate in the alive set.
4. Sends a single `EntropyEvent::SetCells(alive_coords)`.
5. If `reseed_interval` is `Some(duration)`, sleeps for `duration` and repeats from step 1. Each reseed generates a `ClearRegion` covering the full grid followed by a new `SetCells`.
6. If `reseed_interval` is `None`, the task exits after the first injection.

### RLE Parser

The RLE format is the de facto standard for Game of Life pattern exchange. Structure:

```
#N Name
#O Author
#C Comment lines
x = width, y = height, rule = B3/S23
run-length encoded body
```

Parsing steps:

1. Split input into lines.
2. Lines starting with `#` are metadata: `#N` = name, `#O` = author, `#C` = comment, `#R` = rule (legacy).
3. The header line matches `x = <int>, y = <int>` with optional `rule = <rulestring>`.
4. The body is a stream of tokens: digits (run count), `b` (dead cells), `o` (alive cells), `$` (end of row), `!` (end of pattern). Whitespace and newlines within the body are ignored.
5. Default run count is 1 when no digit precedes a cell character.
6. Coordinates are emitted as 2D `[row, col]` vectors. The parser does not assume dimensionality beyond 2D for the file format itself, but the output `Vec<Vec<usize>>` is dimension-agnostic.

Error cases: missing header line, invalid run-length encoding, file exceeds 1 MB (NFR-1).

### Life 1.06 Parser

Life 1.06 is a simpler format:

```
#Life 1.06
x1 y1
x2 y2
...
```

Parsing steps:

1. First line must be `#Life 1.06` (exact match, case-sensitive).
2. Subsequent non-empty lines are space-separated integer pairs: `x y`.
3. Coordinates may be negative (the format allows it). The parser normalizes coordinates by finding the minimum x and y and shifting all coordinates so the minimum is 0.
4. Output is `Vec<Vec<usize>>` with 2D coordinates after normalization.

### GliderGunSource Behavior

The Gosper glider gun is a 36x9 pattern that produces a stream of gliders. The source:

1. Stores the gun pattern as a constant coordinate list (hardcoded from the well-known pattern).
2. On each interval tick, picks a random position within the grid bounds (ensuring the 36x9 pattern fits).
3. Sends `EntropyEvent::ApplyPattern { cells: gun_coords, offset: random_position }`.
4. The gun, once placed, naturally produces gliders through the simulation rules. The source does not simulate the gun; it just places the static pattern and lets the engine do the rest.

### NoiseSource Behavior

Uses the `noise` crate for coherent noise generation:

1. Creates a `Perlin` or `Simplex` noise generator (seeded from system entropy).
2. Maintains a time offset `t` that advances by `scroll_speed * elapsed_seconds` each tick.
3. On each tick, evaluates noise at every grid coordinate: `noise.get([x * scale, y * scale, t])`.
4. Noise output is in `[-1.0, 1.0]`. Map to `[0.0, 1.0]` via `(value + 1.0) / 2.0`.
5. Coordinates where the mapped value exceeds `threshold` are alive; others are dead.
6. Sends a `ClearRegion` for the full grid followed by `SetCells` with alive coordinates.
7. The third dimension `t` in the noise function creates the scrolling effect -- the pattern smoothly evolves over time without any random jumps.

### PatternLibrary

A compile-time `HashMap` (initialized via `once_cell::sync::Lazy` or `std::sync::LazyLock`) containing named patterns as coordinate lists. Bundled patterns:

| Name | Size | Description |
|------|------|-------------|
| `glider` | 3x3 | The classic 5-cell diagonal glider |
| `lwss` | 5x4 | Lightweight spaceship |
| `mwss` | 6x5 | Middleweight spaceship |
| `hwss` | 7x5 | Heavyweight spaceship |
| `pulsar` | 13x13 | Period-3 oscillator |
| `pentadecathlon` | 16x3 | Period-15 oscillator |
| `r-pentomino` | 3x3 | Methuselah that evolves for 1103 generations |
| `gosper-glider-gun` | 36x9 | The Gosper glider gun (shared with GliderGunSource) |
| `beacon` | 4x4 | Period-2 oscillator |
| `toad` | 4x4 | Period-2 oscillator |
| `block` | 2x2 | Still life |
| `beehive` | 4x3 | Still life |
| `loaf` | 4x4 | Still life |
| `boat` | 3x3 | Still life |

Coordinates are stored relative to `[0, 0]`. The `to_event` method adds the caller-provided offset to each coordinate, producing an `ApplyPattern` event.

### File Loading as One-Shot Source

Loading a pattern from an RLE or Life 1.06 file is not a continuous entropy source -- it is a one-shot injection. Two approaches:

**Chosen approach:** Provide a `FilePatternSource` that wraps any `ParsedPattern` as an `EntropySource`. On `start()`, it sends a single `ApplyPattern` event and the background task exits. This keeps the source registry as the single interface for all pattern injection, including file loads. The CLI can do:

```rust
let pattern = RleParser::parse_file("gosper.rle")?;
let source = FilePatternSource::new(pattern, offset);
registry.register(Box::new(source));
```

```rust
pub struct FilePatternSource {
    pattern: ParsedPattern,
    offset: Vec<usize>,
    tx: Option<mpsc::Sender<EntropyEvent>>,
    task: Option<JoinHandle<()>>,
}

impl FilePatternSource {
    pub fn new(pattern: ParsedPattern, offset: Vec<usize>) -> Self;
}

impl EntropySource for FilePatternSource {
    fn name(&self) -> &str;       // pattern.name or "file-pattern"
    fn kind(&self) -> &str;       // "classic"
    // ...
}
```

## Data Model

```
RandomConfig
  density: f64          -- 0.0..=1.0
  reseed_interval: Option<Duration>
  dimensions: Vec<usize>

ParsedPattern
  name: Option<String>
  author: Option<String>
  comments: Vec<String>
  rule: Option<String>
  alive_cells: Vec<Vec<usize>>  -- each entry is one alive cell coordinate
  bounding_box: Vec<usize>      -- extents of the pattern

PatternLibrary
  patterns: HashMap<String, Vec<Vec<usize>>>  -- name -> relative coordinates

NoiseConfig
  noise_type: NoiseType (Perlin | Simplex)
  scale: f64
  threshold: f64
  scroll_speed: f64
  tick_interval: Duration
  dimensions: Vec<usize>
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Malformed RLE files cause parser panics | Source of crashes from user-provided files | Medium | Return `Result` from all parse methods. Validate header values. Limit input size to 1 MB (NFR-1). Fuzz test the parser. |
| Noise source full-grid evaluation is expensive at large grid sizes | Frame drops or event backpressure on large grids | Medium | The noise tick interval is configurable. For large grids, recommend longer intervals. The rate limiter in the registry provides a secondary throttle. |
| Pattern coordinates exceed grid bounds when placed with offset | Engine receives out-of-bounds coordinates | Low | The simulation engine (SPEC-001) already validates coordinates on event application. Sources should clip or wrap, but the engine is the safety net. |
| The `noise` crate adds a non-trivial dependency | Increased compile time and binary size | Low | The `noise` crate is well-maintained and has no transitive dependencies beyond `std`. The compile cost is acceptable for the visual quality it provides. |
| Hardcoded pattern coordinates in PatternLibrary may have transcription errors | Patterns behave incorrectly | Medium | Verify each pattern against a known reference (LifeWiki). Unit tests run each pattern for a few generations and verify expected population counts. |

## ADR-001: Parsers in golf-entropy vs golf-formats

**Status:** Accepted

**Context:** The workspace layout in SPEC.md lists `golf-formats` as a separate crate for file format parsers. However, the parsers produce `ParsedPattern` which is consumed exclusively by entropy sources in `golf-entropy`. A separate crate adds a dependency edge and crate boundary for code that has exactly one consumer.

**Decision:** Place parsers in `golf-entropy` under `src/sources/rle.rs` and `src/sources/life106.rs`. The `ParsedPattern` type lives in `golf-entropy`. If a future spec (e.g., pattern export or editor) needs the parsers independently, extract `golf-formats` at that time.

**Consequences:**
- Fewer crates, simpler dependency graph, faster compilation.
- The `golf-formats` crate in the workspace layout becomes a placeholder for future needs.
- If extraction is needed later, it is a mechanical refactor (move files, update imports) with no behavioral change.

## ADR-002: Full-Grid Clear+Set for Noise vs Differential Updates

**Status:** Accepted

**Context:** The noise source can either send the full alive set each tick (clear + set) or compute the diff between the previous and current state and send only changes. Differential updates are more efficient on the wire but require the source to track previous state.

**Decision:** Use clear+set (full replacement) each tick.

**Consequences:**
- Simpler implementation: no previous-state tracking in the source.
- Higher event volume per tick (one `ClearRegion` + one `SetCells` with potentially thousands of coordinates).
- Acceptable because the noise tick interval is configurable and the rate limiter bounds the events per drain cycle. For a 200x50 grid, the worst case is ~10,000 coordinates per tick, which is a ~80 KB allocation -- well within budget.
- If profiling shows this is a bottleneck, switch to differential updates in a future patch.
