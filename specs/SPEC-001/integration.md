# SPEC-001: Core Engine — Integration

## Interfaces Provided

### NdGrid<N> (Type)

- **Consumers:** SPEC-002 (Rendering), SPEC-003 (Entropy Trait), SPEC-004 (Classic Sources), SPEC-009 (Visual Effects), SPEC-011 (MCP Interface)
- **Contract:** `NdGrid<N>` exposes `cells() -> &[Cell]`, `extents() -> [usize; N]`, `get(coord) -> Cell`. Read-only access to grid state for rendering and inspection.
- **Stability:** Core type. Field layout is private; access is through methods only. Method signatures are stable after SPEC-001 reaches `done`.

### Cell Type (u16)

- **Consumers:** SPEC-002 (Rendering — age-to-color mapping), SPEC-003 (Entropy Trait — event payloads), SPEC-009 (Visual Effects — trail/fade calculations)
- **Contract:** `type Cell = u16`. Value `0` = dead. Value `>0` = age (generation count since birth). Age increments with `saturating_add`.
- **Stability:** Type alias is stable. Semantic meaning (0=dead, >0=age) is a public contract.

### RuleSet and NeighborhoodKind (Types)

- **Consumers:** SPEC-003 (Entropy Trait — `MutateRules` event), SPEC-008 (AI Integration — rule evolution), SPEC-010 (CLI — rule selection), SPEC-011 (MCP Interface — rule queries/mutations)
- **Contract:** `RuleSet { birth: BTreeSet<u32>, survival: BTreeSet<u32>, neighborhood: NeighborhoodKind }`. `NeighborhoodKind` is `Moore | VonNeumann`.
- **Stability:** Struct fields are public. Adding a new `NeighborhoodKind` variant is a breaking change gated by a new spec.

### Preset (Enum)

- **Consumers:** SPEC-010 (CLI — `--preset conway`), SPEC-011 (MCP Interface)
- **Contract:** `Preset::Conway`, `Preset::HighLife`, `Preset::DayNight`, `Preset::ThreeDLife`, `Preset::Custom(RuleSet)`. `RuleSet::from_preset(preset) -> RuleSet` converts.
- **Stability:** New variants may be added in minor versions. `Custom` variant is the escape hatch.

### EdgeBehavior (Enum)

- **Consumers:** SPEC-010 (CLI — edge config), SPEC-011 (MCP Interface)
- **Contract:** `EdgeBehavior::Toroidal | EdgeBehavior::Bounded`. Per-dimension configuration via `[EdgeBehavior; N]`.
- **Stability:** New variants (e.g., `Reflective`) may be added in future specs.

### EntropyEvent<N> (Enum)

- **Consumers:** SPEC-003 (Entropy Trait — sources produce these), SPEC-004 (Classic Sources — pattern loading), SPEC-005 (GPU Entropy), SPEC-006 (Audio Entropy), SPEC-007 (Network Entropy), SPEC-008 (AI Integration — pattern generation), SPEC-011 (MCP Interface — event injection)
- **Contract:** `EntropyEvent::SetCells { coords, state }`, `EntropyEvent::ClearRegion { origin, extents }`, `EntropyEvent::MutateRules(RuleSet)`. Future specs may propose new variants.
- **Stability:** Enum is `#[non_exhaustive]` to allow new variants without a major version bump.

### SimulationEngine<N> (Struct)

- **Consumers:** SPEC-002 (Rendering — reads grid state), SPEC-003 (Entropy Trait — injects events), SPEC-010 (CLI — constructs and runs engine), SPEC-011 (MCP Interface — controls simulation)
- **Contract:**
  - `new(extents, rules, edges) -> Self` — construction
  - `inject_events(events)` — apply entropy events before next step
  - `step()` — advance one generation
  - `grid() -> &NdGrid<N>` — read current state
  - `generation() -> u64` — current generation count
  - `rules() -> &RuleSet` — current rules
  - `set_rules(rules)` — replace rules
- **Stability:** Method signatures are stable after SPEC-001 reaches `done`. Internal fields are private.

## Interfaces Consumed

None. SPEC-001 is the foundation crate with zero external spec dependencies. It depends only on Rust `std` and `alloc`.

## Shared State

SPEC-001 does not introduce any global or shared mutable state. `SimulationEngine<N>` is owned by a single caller (the main loop or CLI harness from SPEC-010). Thread safety, if needed, is the responsibility of the consumer — the engine is `Send` but not `Sync`.

The double buffer is entirely internal to `NdGrid<N>`. Consumers never see or interact with the back buffer.

## Event Flows

### Simulation Step Cycle

```
1. Caller collects EntropyEvent<N> values (from channels, files, user input)
2. Caller calls engine.inject_events(events)
   -> Events applied to front buffer in order
   -> MutateRules triggers neighbor offset recomputation
3. Caller calls engine.step()
   -> For each cell: count neighbors from front buffer, apply rules, write to back buffer
   -> Swap front/back buffers
   -> Increment generation
4. Caller calls engine.grid() to read new state for rendering
```

### Event Application Order

Events are applied in the order they are provided to `inject_events()`. If multiple events target the same cell, the last one wins. `MutateRules` takes effect for the step that immediately follows.

## Cross-Cutting Concerns

### Performance

- The hot path is the `step()` method. All design decisions (flat buffer, precomputed offsets, const generic N) serve this path.
- Consumers must not hold references to grid internals across a `step()` call — the buffer swap invalidates the previous `cells()` slice.
- NFR-1 (under 1ms for 200x50) and NFR-3 (under 50ms for 50x50x50) are verified by benchmarks in SPEC-001.

### Error Handling

- `NdGrid::new()` panics if any extent is 0 or if the total cell count overflows `usize`. These are programming errors, not runtime conditions.
- `inject_events()` silently ignores coordinates that are out of bounds (bounded grids) or wraps them (toroidal grids). No error return — entropy events are best-effort by design.
- `RuleSet::from_preset()` is infallible.

### Serialization

SPEC-001 does not implement serialization. If SPEC-010 or SPEC-011 need to serialize grid state or rules, they will add `serde` derives via feature flags on golf-core types. The types are designed to be serialization-friendly (no function pointers, no self-referential structures).

## Integration Test Requirements

| Test | Validates | Specs Involved |
|------|-----------|----------------|
| Glider translation on 2D toroidal grid | AC-1, AC-2 | SPEC-001 only |
| 3D Moore neighbor count = 26 | AC-3 | SPEC-001 only |
| Cell age increments on survival | AC-4 | SPEC-001 only |
| SetCells entropy event applies to grid | AC-5 | SPEC-001 (SPEC-003 will add channel-based delivery) |
| HighLife replicator with custom rules | AC-6 | SPEC-001 only |
| Step performance benchmark (200x50, 1000 steps) | AC-7, NFR-1 | SPEC-001 only |
| Step performance benchmark (50x50x50, 100 steps) | NFR-3 | SPEC-001 only |

All integration tests for SPEC-001 are self-contained — no other specs need to be implemented first. Future specs (especially SPEC-003) will add integration tests that exercise the entropy injection path end-to-end with real entropy sources.
