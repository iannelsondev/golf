# SPEC-004: Classic Sources -- Integration

## Interfaces Provided

### RandomSource

- Type: `golf_entropy::sources::RandomSource`
- Contract: Implements `EntropySource`. On `start()`, spawns a tokio task that sends `SetCells` events at the configured density. If `reseed_interval` is set, periodically sends `ClearRegion` + `SetCells`. Reports `name: "random"`, `kind: "classic"`.
- Consumers: SPEC-010 (CLI, `--source random` flag), SPEC-001 (as fallback when no other source is configured)

### RleParser

- Type: `golf_entropy::sources::RleParser`
- Contract: `parse(input: &str) -> Result<ParsedPattern>` and `parse_file(path: &Path) -> Result<ParsedPattern>`. Handles `#N`, `#O`, `#C` comment lines, `x = W, y = H, rule = R` header, and run-length encoded body (`digits + b/o/$/!`). Returns error on malformed input or files exceeding 1 MB.
- Consumers: SPEC-010 (CLI, `--pattern-file` flag), FilePatternSource

### Life106Parser

- Type: `golf_entropy::sources::Life106Parser`
- Contract: `parse(input: &str) -> Result<ParsedPattern>` and `parse_file(path: &Path) -> Result<ParsedPattern>`. Requires `#Life 1.06` header. Parses `x y` integer pairs per line. Normalizes negative coordinates to zero-based.
- Consumers: SPEC-010 (CLI, `--pattern-file` flag with auto-detection), FilePatternSource

### FilePatternSource

- Type: `golf_entropy::sources::FilePatternSource`
- Contract: Wraps a `ParsedPattern` as a one-shot `EntropySource`. On `start()`, sends a single `ApplyPattern` event with the pattern's coordinates at the given offset, then the background task exits. Reports `kind: "classic"`.
- Consumers: SPEC-010 (CLI, file loading flow)

### GliderGunSource

- Type: `golf_entropy::sources::GliderGunSource`
- Contract: Implements `EntropySource`. On each interval tick, sends `ApplyPattern` with a Gosper glider gun placed at a random position within grid bounds. Reports `name: "glider-gun"`, `kind: "classic"`.
- Consumers: SPEC-010 (CLI, `--source glider-gun` flag)

### NoiseSource

- Type: `golf_entropy::sources::NoiseSource`
- Contract: Implements `EntropySource`. On each tick, sends `ClearRegion` (full grid) followed by `SetCells` with coordinates where noise value exceeds threshold. Noise scrolls through the third dimension over time. Reports `name: "noise"`, `kind: "classic"`.
- Consumers: SPEC-010 (CLI, `--source noise` flag)

### PatternLibrary

- Type: `golf_entropy::sources::PatternLibrary`
- Contract: `get(name: &str) -> Option<&Vec<Vec<usize>>>` returns coordinates for a named pattern (case-insensitive lookup). `list() -> Vec<&str>` returns all available names. `to_event(name, offset) -> Option<EntropyEvent>` produces an `ApplyPattern` event. Bundled patterns: glider, lwss, mwss, hwss, pulsar, pentadecathlon, r-pentomino, gosper-glider-gun, beacon, toad, block, beehive, loaf, boat.
- Consumers: SPEC-010 (CLI, `--pattern <name>` flag, `golf list-patterns` subcommand), GliderGunSource (for the gun coordinates)

### ParsedPattern

- Type: `golf_entropy::sources::ParsedPattern`
- Contract: Struct with `name: Option<String>`, `author: Option<String>`, `comments: Vec<String>`, `rule: Option<String>`, `alive_cells: Vec<Vec<usize>>`, `bounding_box: Vec<usize>`. Produced by parsers and PatternLibrary, consumed by FilePatternSource and directly by callers that want to inspect pattern metadata.
- Consumers: FilePatternSource, SPEC-010 (CLI, pattern info display)

## Interfaces Consumed

### SPEC-003: Entropy Trait

- Consumes: `EntropySource` trait, `EntropyEvent` enum, `SourceMetadata` struct, `SourceStatus` enum
- Version: SPEC-003 design.md as of draft status (pre-implementation)
- Contract:
  - `EntropySource` trait with methods: `name() -> &str`, `kind() -> &str`, `start() -> Result<Receiver<EntropyEvent>>`, `stop() -> Result<()>`, `metadata() -> SourceMetadata`
  - `EntropyEvent` variants used by this spec: `SetCells(Vec<Vec<usize>>)`, `ClearRegion { origin, extent }`, `ApplyPattern { cells, offset }`
  - `SourceMetadata` with fields: `name: String`, `kind: String`, `status: SourceStatus`, `events_per_second: f64`
  - `SourceStatus` enum: `Running`, `Stopped`, `Error(String)`
  - Channel model: `start()` returns `tokio::sync::mpsc::Receiver<EntropyEvent>`. Source owns the `Sender`. Bounded channel capacity.
- Note: `MutateRules` and `EnergyPulse` variants are not used by any classic source. Classic sources only produce cell-placement events.

### SPEC-001: Core Engine (indirect)

- Consumes: No direct dependency. Classic sources produce `EntropyEvent` values that the engine applies via the registry drain mechanism (SPEC-003). The engine validates coordinate dimensionality at application time.
- Version: SPEC-001 design.md as of draft status
- Contract: Coordinates in events are `Vec<usize>`. The engine checks `coord.len() == N` for each coordinate when applying to `NdGrid<N>`.

## Event Flows

### Random Source Flow

```
1. CLI creates RandomSource with density=0.3, dimensions=[200,50]
2. CLI calls registry.register(Box::new(random_source))
3. CLI calls registry.start(id)
4. RandomSource.start() spawns tokio task, returns Receiver
5. Task generates coordinates: for each (x,y) in grid, rand < 0.3 -> include
6. Task sends EntropyEvent::SetCells(alive_coords)
7. If reseed_interval is set: task sleeps, then sends ClearRegion + new SetCells
8. Engine calls registry.drain() -> receives SetCells event
9. Engine applies SetCells to NdGrid<2>
```

### File Pattern Flow

```
1. CLI detects --pattern-file gosper.rle
2. CLI calls RleParser::parse_file("gosper.rle") -> ParsedPattern
3. CLI creates FilePatternSource::new(pattern, offset=[100, 25])
4. CLI calls registry.register(Box::new(file_source))
5. CLI calls registry.start(id)
6. FilePatternSource.start() spawns task
7. Task sends EntropyEvent::ApplyPattern { cells, offset }
8. Task exits (one-shot)
9. Engine drains and applies the pattern
```

### Named Pattern Flow

```
1. CLI detects --pattern glider
2. CLI calls PatternLibrary::new().to_event("glider", center_offset)
3. CLI wraps in FilePatternSource (or directly injects via a one-shot source)
4. Same registry flow as file patterns
```

### Noise Source Flow

```
1. CLI creates NoiseSource with scale=0.05, threshold=0.5, scroll_speed=0.1
2. Registered and started via registry
3. Each tick (e.g., every 100ms):
   a. Evaluate noise(x*scale, y*scale, t) for all grid coordinates
   b. Send ClearRegion { origin: [0,0], extent: [200,50] }
   c. Send SetCells(coords where noise > threshold)
   d. Advance t += scroll_speed * tick_interval.as_secs_f64()
4. Engine drains ClearRegion + SetCells each step
```

### Glider Gun Source Flow

```
1. CLI creates GliderGunSource with interval=10s, dimensions=[200,50]
2. Registered and started via registry
3. Every 10 seconds:
   a. Pick random (x, y) such that x+36 <= 200 and y+9 <= 50
   b. Fetch gun coords from PatternLibrary
   c. Send ApplyPattern { cells: gun_coords, offset: [x, y] }
4. Engine drains and overlays the gun pattern
```

## Crate Dependency Graph

```
golf-core (defines: EntropyEvent, RuleSet, SourceStatus, SourceMetadata)
    ^
    |
golf-entropy (defines: EntropySource trait, SourceRegistry, RateLimitConfig)
    |
    +-- src/sources/ (SPEC-004: RandomSource, parsers, NoiseSource, etc.)
        depends on: rand, noise crate
        depends on: golf-core for EntropyEvent variants
        depends on: golf-entropy for EntropySource trait (same crate)
```

No new crate dependencies beyond `rand` (randomness for RandomSource and GliderGunSource placement) and `noise` (Perlin/simplex for NoiseSource). Both are well-established crates with minimal transitive dependencies.

## Integration Test Requirements

### IT-1: Random source density accuracy

Create a `RandomSource` with `density: 0.3` and `dimensions: [100, 100]`. Start it, drain the `SetCells` event, count alive coordinates. Verify count is within 5% of 3000 (expected: 100*100*0.3 = 3000, tolerance: 2850..3150).

### IT-2: RLE parser round-trip with Gosper glider gun

Parse a known-good RLE string for the Gosper glider gun. Verify `ParsedPattern.alive_cells` contains exactly 36 alive cells. Verify bounding box is `[36, 9]`. Place on a 100x100 grid via `FilePatternSource`, run 100 simulation steps via `SimulationEngine`, verify population has increased (gliders being produced).

### IT-3: Life 1.06 parser with negative coordinates

Parse a Life 1.06 string with negative coordinate values. Verify all output coordinates are non-negative after normalization. Verify relative positions are preserved.

### IT-4: Glider gun periodic injection

Create a `GliderGunSource` with `interval: 100ms` and grid dimensions `[200, 50]`. Start it, wait 250ms, drain all events. Verify at least 2 `ApplyPattern` events were received. Verify pattern offsets are within grid bounds.

### IT-5: Noise source spatial coherence

Create a `NoiseSource` with known seed, `scale: 0.1`, `threshold: 0.5`. Drain one tick of events. Verify that alive cells form spatially coherent clusters (adjacent alive cells exist), not a random scatter. Heuristic: count the fraction of alive cells that have at least one alive neighbor; it should exceed 50% (random would be ~density).

### IT-6: Pattern library completeness

Instantiate `PatternLibrary`. Verify `list()` contains at least 14 entries. For each pattern, verify `get(name)` returns `Some` with non-empty coordinates. Verify `to_event(name, [0,0])` returns `Some(EntropyEvent::ApplyPattern { .. })`.

### IT-7: Pattern library correctness -- glider evolution

Load the "glider" pattern from `PatternLibrary`. Place on a 10x10 grid. Run 4 simulation steps. Verify the glider has moved one cell diagonally (expected behavior of a glider after 4 generations).

### IT-8: File pattern source one-shot behavior

Create a `FilePatternSource` from a simple 3-cell pattern. Start it. Drain events. Verify exactly one `ApplyPattern` event. Wait 100ms, drain again. Verify zero additional events (task exited).
