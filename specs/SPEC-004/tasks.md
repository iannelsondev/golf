# SPEC-004: Classic Sources -- Tasks

## Task Order

Tasks are ordered by dependency. Each task lists the FR(s) and AC(s) it satisfies.

---

### T-1: Define ParsedPattern type and PatternLibrary

**FR:** FR-6
**AC:** AC-6
**Depends on:** SPEC-003 T-1 (golf-entropy crate exists)

Create `crates/golf-entropy/src/sources/mod.rs` with module declarations. Define `ParsedPattern` struct in `crates/golf-entropy/src/sources/pattern_library.rs`. Implement `PatternLibrary` with all 14 bundled patterns (glider, lwss, mwss, hwss, pulsar, pentadecathlon, r-pentomino, gosper-glider-gun, beacon, toad, block, beehive, loaf, boat). Implement `new()`, `get()`, `list()`, `to_pattern()`, `to_event()`.

**Deliverables:**
- `crates/golf-entropy/src/sources/mod.rs`
- `crates/golf-entropy/src/sources/pattern_library.rs`
- Unit tests: `list()` returns 14 entries, `get("glider")` returns 5 coordinates, `get("nonexistent")` returns `None`, `to_event` produces `ApplyPattern` variant, case-insensitive lookup works

---

### T-2: Implement RleParser

**FR:** FR-2
**AC:** AC-2
**Depends on:** T-1 (ParsedPattern type)

Implement `RleParser` in `crates/golf-entropy/src/sources/rle.rs`. Handle `#N`, `#O`, `#C` comment lines, `x = W, y = H, rule = R` header, and run-length encoded body. Reject files exceeding 1 MB (NFR-1). `parse()` and `parse_file()` both return `Result<ParsedPattern>`.

**Deliverables:**
- `crates/golf-entropy/src/sources/rle.rs`
- Unit tests:
  - Parse a minimal RLE string (`x = 3, y = 3\nbo$2bo$3o!`) into correct coordinates
  - Parse RLE with all metadata lines (`#N`, `#O`, `#C`) and verify fields populated
  - Parse Gosper glider gun RLE, verify 36 alive cells
  - Error on missing header line
  - Error on input exceeding 1 MB
  - Handle multi-digit run counts (e.g., `10b3o`)
  - Handle `$` row separators and multiple `$` for blank rows

---

### T-3: Implement Life106Parser

**FR:** FR-3
**AC:** AC-3
**Depends on:** T-1 (ParsedPattern type)

Implement `Life106Parser` in `crates/golf-entropy/src/sources/life106.rs`. Require `#Life 1.06` header. Parse `x y` integer pairs. Normalize negative coordinates to zero-based. `parse()` and `parse_file()` return `Result<ParsedPattern>`.

**Deliverables:**
- `crates/golf-entropy/src/sources/life106.rs`
- Unit tests:
  - Parse valid Life 1.06 with positive coordinates
  - Parse with negative coordinates, verify normalization shifts minimum to 0
  - Error on missing `#Life 1.06` header
  - Skip blank lines and whitespace
  - Error on non-integer coordinate values

---

### T-4: Implement FilePatternSource

**FR:** FR-2, FR-3
**AC:** AC-2, AC-3
**Depends on:** T-1, T-2, T-3

Implement `FilePatternSource` in `crates/golf-entropy/src/sources/mod.rs` (or a dedicated file). Wraps a `ParsedPattern` as a one-shot `EntropySource`. On `start()`, spawns a tokio task that sends a single `ApplyPattern` event and exits.

**Deliverables:**
- `FilePatternSource` implementation with `EntropySource` trait
- Unit tests:
  - Start source, drain one `ApplyPattern` event with correct coordinates and offset
  - After drain, no further events (task exited)
  - `metadata()` reports correct name and kind

---

### T-5: Implement RandomSource

**FR:** FR-1
**AC:** AC-1
**Depends on:** T-1 (sources module exists), SPEC-003 (EntropySource trait)

Implement `RandomSource` in `crates/golf-entropy/src/sources/random.rs`. Support configurable density and optional reseed interval. On start, spawn tokio task that generates coordinates based on density and sends `SetCells`. If reseed interval is set, loop with `tokio::time::interval`.

**Deliverables:**
- `crates/golf-entropy/src/sources/random.rs`
- Unit tests:
  - Density 0.0 produces zero alive cells
  - Density 1.0 produces all cells alive
  - Density 0.5 on a 100x100 grid produces ~5000 cells (within 10% tolerance)
  - Without reseed: one event, then task exits
  - With reseed interval: multiple events over time
- Add `rand` dependency to `crates/golf-entropy/Cargo.toml`

---

### T-6: Implement GliderGunSource

**FR:** FR-4
**AC:** AC-4
**Depends on:** T-1 (PatternLibrary for gun coordinates), T-5 (RandomSource pattern for periodic source)

Implement `GliderGunSource` in `crates/golf-entropy/src/sources/glider_gun.rs`. On each interval tick, fetch Gosper glider gun coordinates from `PatternLibrary`, pick a random offset within grid bounds, send `ApplyPattern`.

**Deliverables:**
- `crates/golf-entropy/src/sources/glider_gun.rs`
- Unit tests:
  - Start with 100ms interval, wait 250ms, verify at least 2 `ApplyPattern` events
  - All pattern offsets are within grid bounds (offset + 36 <= width, offset + 9 <= height)
  - Stop halts further events

---

### T-7: Implement NoiseSource

**FR:** FR-5
**AC:** AC-5
**Depends on:** T-1 (sources module exists), SPEC-003 (EntropySource trait)

Implement `NoiseSource` in `crates/golf-entropy/src/sources/noise.rs`. Use the `noise` crate for Perlin or Simplex generation. On each tick, evaluate noise across all grid coordinates, send `ClearRegion` + `SetCells` for coordinates exceeding threshold.

**Deliverables:**
- `crates/golf-entropy/src/sources/noise.rs`
- Unit tests:
  - Threshold 0.0 produces all cells alive (any positive noise passes)
  - Threshold 1.0 produces zero cells alive (nothing exceeds 1.0)
  - Spatial coherence check: alive cells have neighbor density above random expectation
  - Time dimension advances: two consecutive ticks produce different alive sets
- Add `noise` dependency to `crates/golf-entropy/Cargo.toml`

---

### T-8: Wire up module re-exports and update Cargo.toml

**FR:** all (public interface)
**AC:** all (usability)
**Depends on:** T-1 through T-7

Update `crates/golf-entropy/src/sources/mod.rs` to re-export all public types: `RandomSource`, `RandomConfig`, `RleParser`, `Life106Parser`, `FilePatternSource`, `GliderGunSource`, `GliderGunConfig`, `NoiseSource`, `NoiseConfig`, `NoiseType`, `PatternLibrary`, `ParsedPattern`. Update `crates/golf-entropy/src/lib.rs` to re-export the `sources` module.

**Deliverables:**
- `crates/golf-entropy/src/sources/mod.rs` updated
- `crates/golf-entropy/src/lib.rs` updated
- Compile check: `cargo check -p golf-entropy`

---

### T-9: Integration tests

**FR:** FR-1 through FR-6
**AC:** AC-1 through AC-6
**Depends on:** T-8

Write integration tests in `crates/golf-entropy/tests/classic_sources.rs` covering IT-1 through IT-8 from integration.md. Tests run with `#[tokio::test]`.

**Deliverables:**
- `crates/golf-entropy/tests/classic_sources.rs`
- All 8 integration test scenarios passing

---

### T-10: Pattern load performance benchmark (NFR-2)

**FR:** none (non-functional)
**AC:** none (NFR-2 validation)
**Depends on:** T-1, T-2

Add a criterion benchmark in `crates/golf-entropy/benches/pattern_load.rs`. Benchmark `PatternLibrary::get()` for each pattern and `RleParser::parse()` for a Gosper glider gun RLE string. Assert all complete in under 10ms.

**Deliverables:**
- `crates/golf-entropy/benches/pattern_load.rs`
- `Cargo.toml` updated with `[[bench]]` section if not already present

---

## Task Dependency Graph

```
T-1 (ParsedPattern + PatternLibrary)
 |
 +---> T-2 (RleParser) --------+---> T-4 (FilePatternSource) --+
 |                              |                                |
 +---> T-3 (Life106Parser) ----+                                |
 |                                                               |
 +---> T-5 (RandomSource) -------------------------------------+
 |                                                               |
 +---> T-6 (GliderGunSource) ----------------------------------+
 |                                                               |
 +---> T-7 (NoiseSource) --------------------------------------+
                                                                 |
                                                          T-8 (re-exports)
                                                                 |
                                                       +---------+---------+
                                                       |                   |
                                                 T-9 (integration)   T-10 (bench)
```

## FR-to-Task Mapping

| FR | Tasks |
|----|-------|
| FR-1 (Random source) | T-5, T-9 |
| FR-2 (RLE parser) | T-2, T-4, T-9 |
| FR-3 (Life 1.06 parser) | T-3, T-4, T-9 |
| FR-4 (Glider gun source) | T-6, T-9 |
| FR-5 (Noise source) | T-7, T-9 |
| FR-6 (Pattern library) | T-1, T-9 |

## AC-to-Task Mapping

| AC | Primary Task | Verified In |
|----|-------------|-------------|
| AC-1 (random density 0.3 within 5%) | T-5 | T-9 (IT-1) |
| AC-2 (RLE glider gun loads correctly) | T-2, T-4 | T-9 (IT-2) |
| AC-3 (Life 1.06 coordinate match) | T-3, T-4 | T-9 (IT-3) |
| AC-4 (glider gun injection at interval) | T-6 | T-9 (IT-4) |
| AC-5 (noise smooth spatial gradient) | T-7 | T-9 (IT-5) |
| AC-6 (named pattern placement) | T-1 | T-9 (IT-6, IT-7) |
