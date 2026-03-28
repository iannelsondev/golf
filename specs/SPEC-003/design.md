# SPEC-003: Entropy Trait -- Design

## Signatures

### Files Created or Modified

- `crates/golf-entropy/src/lib.rs` -- crate root, re-exports public API
- `crates/golf-entropy/src/event.rs` -- `EntropyEvent` enum and associated types
- `crates/golf-entropy/src/source.rs` -- `EntropySource` trait definition
- `crates/golf-entropy/src/registry.rs` -- `SourceRegistry` for runtime source management
- `crates/golf-entropy/src/rate_limit.rs` -- per-source rate limiter
- `crates/golf-entropy/Cargo.toml` -- crate manifest

### Types Introduced

- `EntropyEvent` -- enum representing all entropy injections into the simulation
  ```rust
  pub enum EntropyEvent {
      SetCells(Vec<Vec<usize>>),
      ClearRegion { origin: Vec<usize>, extent: Vec<usize> },
      ApplyPattern { cells: Vec<Vec<usize>>, offset: Vec<usize> },
      MutateRules(RuleSet),
      EnergyPulse { center: Vec<usize>, radius: f64, intensity: f64 },
  }
  ```
- `SourceStatus` -- enum for source lifecycle state
  ```rust
  pub enum SourceStatus {
      Stopped,
      Running,
      Error(String),
  }
  ```
- `SourceMetadata` -- runtime metadata reported by each source
  ```rust
  pub struct SourceMetadata {
      pub name: String,
      pub kind: String,
      pub status: SourceStatus,
      pub events_per_second: f64,
  }
  ```
- `SourceId` -- newtype wrapper for source identification
  ```rust
  pub struct SourceId(pub u64);
  ```
- `RateLimitConfig` -- per-source rate limiting configuration
  ```rust
  pub struct RateLimitConfig {
      pub max_events_per_drain: usize,
  }
  ```

### Traits

- `EntropySource` -- async trait that all entropy sources implement
  ```rust
  #[async_trait]
  pub trait EntropySource: Send + Sync {
      fn name(&self) -> &str;
      fn kind(&self) -> &str;
      async fn start(&mut self) -> Result<tokio::sync::mpsc::Receiver<EntropyEvent>>;
      async fn stop(&mut self) -> Result<()>;
      fn metadata(&self) -> SourceMetadata;
  }
  ```

### Functions / Structs with Key Methods

- `SourceRegistry`
  ```rust
  impl SourceRegistry {
      pub fn new() -> Self;
      pub fn register(&mut self, source: Box<dyn EntropySource>) -> SourceId;
      pub fn unregister(&mut self, id: SourceId) -> Result<()>;
      pub async fn start(&mut self, id: SourceId) -> Result<()>;
      pub async fn stop(&mut self, id: SourceId) -> Result<()>;
      pub fn list(&self) -> Vec<(SourceId, SourceMetadata)>;
      pub fn set_rate_limit(&mut self, id: SourceId, config: RateLimitConfig);
      pub fn drain(&mut self, budget: Option<usize>) -> Vec<EntropyEvent>;
  }
  ```
- `drain_with_limit(rx: &mut Receiver<EntropyEvent>, max: usize) -> Vec<EntropyEvent>` -- non-async drain helper that pulls up to `max` events from a receiver using `try_recv`

## Architecture

### Channel Model

Each `EntropySource` owns the sender half of a `tokio::sync::mpsc` channel. When `start()` is called, the source spawns its background task (tokio task, thread, or async stream) and returns the `Receiver<EntropyEvent>` to the caller. The `SourceRegistry` holds all active receivers.

```
Source A (background task) --tx--> [mpsc channel A] --rx--> SourceRegistry
Source B (background task) --tx--> [mpsc channel B] --rx--> SourceRegistry
                                                               |
                                                          drain()
                                                               |
                                                               v
                                                     Vec<EntropyEvent>
                                                   (fed to simulation engine)
```

The simulation engine calls `registry.drain()` once per step. The registry iterates over all active receivers, calling `try_recv` in a loop up to each source's rate limit. Events are collected into a single `Vec<EntropyEvent>` and returned. No async `.await` in the drain path -- `try_recv` is synchronous and non-blocking.

### Dimension-Agnostic Coordinates

Coordinates in `EntropyEvent` use `Vec<usize>` rather than `[usize; N]` const generics. This is a deliberate trade-off:

- Events cross source boundaries and flow through a single channel. Const generic `N` would require the channel, registry, and every source to be generic over `N`, which propagates a compile-time dimension parameter through the entire entropy subsystem.
- Sources like audio or network entropy do not inherently know the grid dimensionality. They produce coordinate data that gets mapped to whatever grid is active.
- The simulation engine validates coordinate dimensionality when applying events to the `NdGrid<N>`. A `Vec<usize>` with wrong length is an error at application time, not at channel time.
- The performance cost of `Vec<usize>` allocation is negligible relative to the entropy source I/O that produces the events.

### Rate Limiting Strategy

Rate limiting is applied at drain time, not at the source. Sources produce events freely into their channels. When `drain()` runs, it reads at most `max_events_per_drain` from each source's receiver. Excess events remain buffered in the channel until the next drain cycle. If the channel fills (bounded capacity), the source's `send()` will apply backpressure or the source can choose to drop events.

Channel capacity is set to `max_events_per_drain * 4` by default, giving sources a 4-cycle buffer before backpressure kicks in.

### Source Lifecycle

```
             register()       start()         stop()        unregister()
  [absent] ----------> [Stopped] ------> [Running] ------> [Stopped] ---------> [absent]
                           ^                  |
                           |    stop()        |
                           +------------------+
                                              |
                                         (on error)
                                              |
                                              v
                                        [Error(msg)]
                                              |
                                         stop() then start()
                                              |
                                              v
                                          [Running]
```

Sources in `Error` state can be stopped and restarted. The registry does not automatically restart failed sources -- that is a policy decision for higher-level code (CLI, config).

### ADR: Vec<usize> over const generic N for event coordinates

**Status:** Accepted

**Context:** SPEC-001 defines `NdGrid<const N: usize>` with compile-time dimensionality. The entropy subsystem needs to pass coordinates through channels and a registry that serves any grid dimensionality.

**Decision:** Use `Vec<usize>` for all coordinates in `EntropyEvent`. Validate dimensionality at the point where events are applied to a concrete `NdGrid<N>`.

**Consequences:**
- Positive: The entire entropy crate is dimension-agnostic. Sources, channels, and the registry do not need generic parameters. A single binary can switch grid dimensions at runtime without recompiling entropy sources.
- Negative: One small heap allocation per coordinate vector. Dimension mismatches are runtime errors instead of compile-time errors. The simulation engine must validate every event's coordinate length.
- Mitigation: Coordinate vectors are small (2-4 elements typically). The engine's validation is a single length check per event, which is cheap relative to grid mutation.

### ADR: Receiver-per-source over shared channel

**Status:** Accepted

**Context:** Events from multiple sources must reach the simulation engine. Two designs: (a) all sources share one channel, (b) each source gets its own channel.

**Decision:** Each source gets its own `mpsc` channel. The registry holds all receivers and round-robins during drain.

**Consequences:**
- Positive: Per-source rate limiting is trivial (drain N from each receiver). Per-source backpressure is independent. A misbehaving source cannot starve others. Source metadata (events/sec) is naturally per-channel.
- Negative: Slightly more complex drain loop (iterate receivers vs. drain one). More channel allocations (one per source vs. one total).
- Mitigation: Source count is small (typically 1-5). The drain loop is a tight `try_recv` loop with no allocation.
