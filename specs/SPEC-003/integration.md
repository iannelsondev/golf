# SPEC-003: Entropy Trait -- Integration

## Interfaces Provided

SPEC-003 provides the foundational entropy abstraction that all entropy source specs (SPEC-004 through SPEC-008) and the simulation engine (SPEC-001) depend on.

### EntropySource trait

- Trait: `golf_entropy::EntropySource`
- Contract: Implementors produce `EntropyEvent` values through a `tokio::sync::mpsc::Receiver` returned from `start()`. Sources must transition to `Stopped` status after `stop()` is called and must not send further events after stopping.
- Consumers: SPEC-004 (Classic Sources), SPEC-005 (GPU Entropy), SPEC-006 (Audio Entropy), SPEC-007 (Network Entropy), SPEC-008 (AI Integration)

### EntropyEvent enum

- Type: `golf_entropy::EntropyEvent`
- Contract: Enum with five variants -- `SetCells`, `ClearRegion`, `ApplyPattern`, `MutateRules`, `EnergyPulse`. Coordinates are `Vec<usize>` (dimension-agnostic). The simulation engine validates coordinate dimensionality at application time.
- Consumers: SPEC-001 (Core Engine, FR-6: accept EntropyEvent injections between steps), SPEC-010 (CLI, for event logging/stats)

### SourceRegistry

- Type: `golf_entropy::SourceRegistry`
- Contract: Register, start, stop, unregister sources at runtime. `drain()` returns a `Vec<EntropyEvent>` with per-source rate limiting applied. Adding/removing sources does not pause the simulation (NFR-2).
- Consumers: SPEC-001 (Core Engine, simulation loop calls `drain()` between steps), SPEC-010 (CLI, `golf list-sources` subcommand)

### SourceMetadata and SourceStatus

- Types: `golf_entropy::SourceMetadata`, `golf_entropy::SourceStatus`
- Contract: `SourceMetadata` contains `name: String`, `kind: String`, `status: SourceStatus`, `events_per_second: f64`. `SourceStatus` is `Running | Stopped | Error(String)`.
- Consumers: SPEC-002 (Rendering, stats overlay), SPEC-010 (CLI, source listing)

### RateLimitConfig

- Type: `golf_entropy::RateLimitConfig`
- Contract: `max_events_per_drain: usize`. Applied per-source during `drain()`. Events beyond the limit remain buffered in the channel.
- Consumers: SPEC-010 (CLI, `--rate-limit` flag)

## Interfaces Consumed

### SPEC-001: Core Engine

- Consumes: `golf_core::RuleSet` struct
- Version: SPEC-001 spec.md as of draft status (pre-implementation)
- Contract: `RuleSet` must include `birth: BTreeSet<u32>`, `survival: BTreeSet<u32>`, `neighborhood: NeighborhoodKind`. Used in `EntropyEvent::MutateRules(RuleSet)` to allow entropy sources to modify simulation rules at runtime.
- Note: `NdGrid<N>` is NOT directly consumed by golf-entropy. The entropy crate produces events; the core engine applies them to the grid. This keeps golf-entropy independent of the grid's const generic `N`.

## Event Flows

### Source to Engine (primary data flow)

```
1. Source background task produces data (GPU read, audio sample, network packet, etc.)
2. Source maps raw data to EntropyEvent variants
3. Source sends EntropyEvent through its mpsc::Sender
4. [channel buffer -- bounded, per-source]
5. Simulation engine calls registry.drain() between steps
6. Registry iterates receivers, try_recv up to rate limit per source
7. Registry returns Vec<EntropyEvent>
8. Engine applies events to NdGrid<N>:
   - SetCells: set cells alive at given coordinates
   - ClearRegion: set cells dead in the bounding box
   - ApplyPattern: overlay pattern at offset
   - MutateRules: replace active RuleSet
   - EnergyPulse: probabilistically set cells within radius
9. Engine runs next simulation step
```

### Source Lifecycle (control flow)

```
1. CLI or config creates a source implementation (e.g., AudioSource::new(config))
2. CLI calls registry.register(source) -> SourceId
3. CLI calls registry.start(id) -> source.start() spawns background task, returns Receiver
4. Registry stores Receiver, source status becomes Running
5. [simulation runs, drain() collects events]
6. CLI calls registry.stop(id) -> source.stop() signals background task to halt
7. Source task exits, channel closes, status becomes Stopped
8. Optionally: registry.unregister(id) removes source entirely
```

### Backpressure Flow

```
1. Source produces events faster than drain() consumes them
2. Channel buffer fills (capacity = max_events_per_drain * 4)
3. Source's send() blocks (or source drops events if using try_send)
4. On next drain(), rate-limited events are consumed, freeing buffer space
5. Source resumes sending
```

## Crate Dependency Graph

```
golf-entropy
  depends on: tokio (mpsc channels, async trait runtime)
  depends on: golf-core (RuleSet type only -- re-exported or defined in shared types)

golf-core
  depends on: golf-entropy (EntropyEvent type for FR-6 event injection)
```

This creates a circular crate dependency. Resolution: extract shared types (`RuleSet`, `EntropyEvent`, coordinate types) into the `golf-core` crate as the foundational types crate, and have `golf-entropy` depend on `golf-core` for `RuleSet` only. The `EntropyEvent` type lives in `golf-entropy`. The core engine depends on `golf-entropy` for the `EntropyEvent` type it consumes. Dependency direction:

```
golf-core --> golf-entropy (for EntropyEvent)
golf-entropy --> golf-core (for RuleSet)
```

To break the cycle, `RuleSet` and `EntropyEvent` could both live in `golf-core`, making `golf-entropy` depend on `golf-core` but not the reverse. The simulation engine in `golf-core` uses both types directly. The entropy crate provides the trait and registry, importing event/rule types from core.

**Resolved dependency direction:**

```
golf-core (defines: NdGrid, RuleSet, EntropyEvent, SourceStatus, SourceMetadata)
    ^
    |
golf-entropy (defines: EntropySource trait, SourceRegistry, RateLimitConfig)
    depends on golf-core for EntropyEvent, RuleSet, SourceStatus, SourceMetadata
```

This means `EntropyEvent` is defined in `golf-core` and re-exported by `golf-entropy` for convenience. The core engine has no dependency on the entropy crate.

## Integration Test Requirements

### IT-1: Source registration and lifecycle

Register a mock `EntropySource` with the registry. Start it, verify status is `Running`. Stop it, verify status is `Stopped`. Unregister it, verify it no longer appears in `list()`.

### IT-2: Event delivery end-to-end

Create a mock source that sends 10 `SetCells` events on start. Register and start it. Call `drain()`. Verify all 10 events are returned. Verify events contain the expected coordinates.

### IT-3: Rate limiting enforcement

Create a mock source that sends 200 events immediately. Set rate limit to 50. Call `drain()`. Verify exactly 50 events are returned. Call `drain()` again. Verify next 50 are returned.

### IT-4: Multiple source interleaving

Register two mock sources, each producing 20 events. Start both. Call `drain()`. Verify events from both sources are present in the result (total 40 or up to rate limits).

### IT-5: Source stop halts event production

Start a mock source that produces events in a loop. Call `stop()`. Wait briefly. Verify no new events appear on subsequent `drain()` calls after the channel is fully drained.

### IT-6: EnergyPulse probabilistic application

Create a mock source that sends one `EnergyPulse` with `intensity: 1.0` and `radius: 3.0`. Apply to a test grid. Verify all cells within the radius are set alive (intensity 1.0 = 100% probability). Repeat with `intensity: 0.0`, verify no cells are set.

### IT-7: Drain performance (NFR-1)

Pre-fill a channel with 100 events. Time `drain()`. Assert completion in under 100 microseconds. Run on a warm cache with multiple iterations to reduce noise.
