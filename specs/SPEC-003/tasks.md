# SPEC-003: Entropy Trait -- Tasks

## Task Order

Tasks are ordered by dependency. Each task lists the FR(s) and AC(s) it satisfies.

---

### T-1: Create golf-entropy crate scaffold

**FR:** none (infrastructure)
**AC:** none
**Depends on:** SPEC-001 types existing in golf-core (RuleSet)

Create `crates/golf-entropy/Cargo.toml` with dependencies on `golf-core`, `tokio` (features: sync, rt), and `async-trait`. Create `crates/golf-entropy/src/lib.rs` with module declarations. Add `golf-entropy` to workspace `Cargo.toml`.

**Deliverables:**
- `crates/golf-entropy/Cargo.toml`
- `crates/golf-entropy/src/lib.rs`
- Workspace `Cargo.toml` updated

---

### T-2: Define EntropyEvent enum and coordinate types

**FR:** FR-2
**AC:** AC-2, AC-6 (event types used in these acceptance criteria)
**Depends on:** T-1, SPEC-001 RuleSet type

Define `EntropyEvent` in `crates/golf-core/src/event.rs` (lives in golf-core to avoid circular dependency). Variants: `SetCells(Vec<Vec<usize>>)`, `ClearRegion { origin, extent }`, `ApplyPattern { cells, offset }`, `MutateRules(RuleSet)`, `EnergyPulse { center, radius, intensity }`. Re-export from `golf-entropy`.

**Deliverables:**
- `crates/golf-core/src/event.rs`
- Unit tests for event construction and Debug/Clone derives

---

### T-3: Define SourceStatus, SourceMetadata, SourceId types

**FR:** FR-6
**AC:** AC-1 (metadata visible in source list)
**Depends on:** T-1

Define `SourceStatus` enum (`Running`, `Stopped`, `Error(String)`), `SourceMetadata` struct (`name`, `kind`, `status`, `events_per_second`), and `SourceId` newtype in `crates/golf-entropy/src/source.rs`.

**Deliverables:**
- `crates/golf-entropy/src/source.rs` (types portion)
- Unit tests for status transitions and metadata construction

---

### T-4: Define EntropySource trait

**FR:** FR-1, FR-3
**AC:** AC-1, AC-5
**Depends on:** T-2, T-3

Define the `EntropySource` async trait in `crates/golf-entropy/src/source.rs`. Methods: `name() -> &str`, `kind() -> &str`, `start() -> Result<Receiver<EntropyEvent>>`, `stop() -> Result<()>`, `metadata() -> SourceMetadata`. Uses `#[async_trait]`.

**Deliverables:**
- `crates/golf-entropy/src/source.rs` (trait definition)
- A `MockSource` test helper that implements `EntropySource` for use in subsequent tests

---

### T-5: Implement RateLimitConfig and drain helper

**FR:** FR-5
**AC:** AC-3
**Depends on:** T-2

Define `RateLimitConfig` in `crates/golf-entropy/src/rate_limit.rs`. Implement `drain_with_limit(rx: &mut Receiver<EntropyEvent>, max: usize) -> Vec<EntropyEvent>` that calls `try_recv` in a loop up to `max` times.

**Deliverables:**
- `crates/golf-entropy/src/rate_limit.rs`
- Unit tests: drain with limit of 50 from channel containing 200 events returns exactly 50; drain from empty channel returns empty vec; drain from channel with fewer events than limit returns all available

---

### T-6: Implement SourceRegistry

**FR:** FR-4, FR-5
**AC:** AC-1, AC-3, AC-4
**Depends on:** T-3, T-4, T-5

Implement `SourceRegistry` in `crates/golf-entropy/src/registry.rs`. Methods: `new()`, `register()`, `unregister()`, `start()`, `stop()`, `list()`, `set_rate_limit()`, `drain()`. The registry holds a `HashMap<SourceId, RegistryEntry>` where `RegistryEntry` contains the source, its receiver (when running), and its rate limit config.

`drain()` iterates all active receivers, calls `drain_with_limit` for each, collects into a single `Vec<EntropyEvent>`.

**Deliverables:**
- `crates/golf-entropy/src/registry.rs`
- Tests:
  - Register source, verify it appears in `list()`
  - Start source, verify status is Running
  - Stop source, verify status is Stopped, no further events
  - Unregister source, verify removed from `list()`
  - Register two sources, start both, drain interleaves events from both

---

### T-7: Wire up public API and re-exports

**FR:** all (public interface)
**AC:** all (usability)
**Depends on:** T-2 through T-6

Update `crates/golf-entropy/src/lib.rs` to re-export all public types: `EntropySource`, `EntropyEvent`, `SourceRegistry`, `SourceId`, `SourceStatus`, `SourceMetadata`, `RateLimitConfig`. Ensure `golf-entropy` is the single import point for downstream crates.

**Deliverables:**
- `crates/golf-entropy/src/lib.rs` updated with re-exports
- Compile check: `cargo check -p golf-entropy`

---

### T-8: Integration tests

**FR:** FR-1 through FR-6
**AC:** AC-1 through AC-6
**Depends on:** T-6, T-7

Write integration tests in `crates/golf-entropy/tests/integration.rs` covering IT-1 through IT-6 from integration.md. Uses `MockSource` from T-4. Tests run with `#[tokio::test]`.

**Deliverables:**
- `crates/golf-entropy/tests/integration.rs`
- All 6 integration test scenarios passing

---

### T-9: Drain performance benchmark (NFR-1)

**FR:** none (non-functional)
**AC:** none (NFR-1 validation)
**Depends on:** T-6

Add a criterion benchmark in `crates/golf-entropy/benches/drain.rs`. Pre-fill a channel with 100 `SetCells` events, benchmark `drain()`. Assert median is under 100 microseconds.

**Deliverables:**
- `crates/golf-entropy/benches/drain.rs`
- `Cargo.toml` updated with `[[bench]]` section and criterion dev-dependency

---

## Task Dependency Graph

```
T-1 (scaffold)
 |
 +---> T-2 (EntropyEvent) ---+---> T-4 (EntropySource trait) --+
 |                            |                                  |
 +---> T-3 (metadata types) --+---> T-5 (rate limiter) ---------+---> T-6 (registry)
                                                                        |
                                                              T-7 (re-exports)
                                                                        |
                                                              +---------+---------+
                                                              |                   |
                                                        T-8 (integration)   T-9 (bench)
```

## FR-to-Task Mapping

| FR | Tasks |
|----|-------|
| FR-1 (EntropySource trait) | T-4, T-8 |
| FR-2 (EntropyEvent enum) | T-2, T-8 |
| FR-3 (Async channel delivery) | T-4, T-6, T-8 |
| FR-4 (Source registry) | T-6, T-8 |
| FR-5 (Rate limiting) | T-5, T-6, T-8 |
| FR-6 (Source metadata) | T-3, T-4, T-8 |

## AC-to-Task Mapping

| AC | Primary Task | Verified In |
|----|-------------|-------------|
| AC-1 (register and list) | T-6 | T-8 (IT-1) |
| AC-2 (SetCells drain) | T-6 | T-8 (IT-2) |
| AC-3 (rate limit 50/1000) | T-5, T-6 | T-8 (IT-3) |
| AC-4 (two source interleave) | T-6 | T-8 (IT-4) |
| AC-5 (stop halts events) | T-4, T-6 | T-8 (IT-5) |
| AC-6 (EnergyPulse application) | T-2 | T-8 (IT-6) |
