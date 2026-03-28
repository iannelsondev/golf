# SPEC-007: Network Entropy -- Design

## Signatures

### Files Created or Modified

- `crates/golf-entropy/src/network.rs` -- module root, re-exports all network sources
- `crates/golf-entropy/src/network/dns.rs` -- `DnsEntropySource`
- `crates/golf-entropy/src/network/ping.rs` -- `PingEntropySource`
- `crates/golf-entropy/src/network/socket.rs` -- `SocketEntropySource`
- `crates/golf-entropy/src/network/iface.rs` -- `InterfaceStatsSource`
- `crates/golf-entropy/src/network/config.rs` -- shared configuration types
- `crates/golf-entropy/Cargo.toml` -- add `surge-ping`, `tokio` DNS features behind `network` feature gate

### Types Introduced

- `DnsEntropySource` -- monitors DNS resolutions, maps query hashes to cell coordinates
  ```rust
  pub struct DnsEntropySource {
      config: DnsConfig,
      status: SourceStatus,
      cancel: Option<CancellationToken>,
  }
  ```
- `DnsConfig`
  ```rust
  pub struct DnsConfig {
      pub targets: Vec<String>,       // hostnames to resolve periodically
      pub interval: Duration,         // default 2s
  }
  ```
- `PingEntropySource` -- pings targets, maps latency to cell birth density
  ```rust
  pub struct PingEntropySource {
      config: PingConfig,
      status: SourceStatus,
      cancel: Option<CancellationToken>,
  }
  ```
- `PingConfig`
  ```rust
  pub struct PingConfig {
      pub targets: Vec<IpAddr>,       // default: [1.1.1.1, 8.8.8.8, 9.9.9.9]
      pub interval: Duration,         // default 1s, minimum 100ms
      pub timeout: Duration,          // default 3s
  }
  ```
- `SocketEntropySource` -- listens on a UDP socket, hashes incoming payloads to coordinates
  ```rust
  pub struct SocketEntropySource {
      config: SocketConfig,
      status: SourceStatus,
      cancel: Option<CancellationToken>,
  }
  ```
- `SocketConfig`
  ```rust
  pub struct SocketConfig {
      pub bind_addr: SocketAddr,      // default 127.0.0.1:19420
      pub max_packet_size: usize,     // default 1500
  }
  ```
- `InterfaceStatsSource` -- reads `/sys/class/net/*/statistics/`, maps rate-of-change to entropy
  ```rust
  pub struct InterfaceStatsSource {
      config: IfaceConfig,
      status: SourceStatus,
      cancel: Option<CancellationToken>,
      prev_stats: HashMap<String, IfaceSnapshot>,
  }
  ```
- `IfaceConfig`
  ```rust
  pub struct IfaceConfig {
      pub interfaces: Vec<String>,    // empty = all non-loopback interfaces
      pub poll_interval: Duration,    // default 1s
  }
  ```
- `IfaceSnapshot` -- a point-in-time reading of interface counters
  ```rust
  struct IfaceSnapshot {
      rx_bytes: u64,
      tx_bytes: u64,
      rx_packets: u64,
      tx_packets: u64,
      timestamp: Instant,
  }
  ```
- `NetworkConfig` -- top-level config aggregating all network sub-sources
  ```rust
  pub struct NetworkConfig {
      pub dns: Option<DnsConfig>,
      pub ping: Option<PingConfig>,
      pub socket: Option<SocketConfig>,
      pub iface: Option<IfaceConfig>,
  }
  ```

### Functions / Hooks / Contracts

- All four source types implement `EntropySource`:
  - `fn name(&self) -> &str` -- returns `"dns"`, `"ping"`, `"socket"`, `"iface-stats"` respectively
  - `fn kind(&self) -> &str` -- returns `"network"` for all
  - `async fn start(&mut self) -> Result<Receiver<EntropyEvent>>` -- spawns async task, returns receiver
  - `async fn stop(&mut self) -> Result<()>` -- cancels task via `CancellationToken`
  - `fn metadata(&self) -> SourceMetadata`
- `fn hash_to_coordinates(data: &[u8], grid_extents: &[usize]) -> Vec<usize>` -- deterministic hash mapping from arbitrary bytes to valid grid coordinates
- `fn latency_to_cell_count(latency: Duration, baseline: Duration) -> usize` -- maps ping latency to number of cells to birth (higher latency = more cells)
- `fn read_iface_stats(interface: &str) -> Result<IfaceSnapshot>` -- reads counters from `/sys/class/net/{interface}/statistics/`
- `fn delta_to_entropy_density(prev: &IfaceSnapshot, current: &IfaceSnapshot) -> f64` -- computes rate-of-change normalized to 0.0..1.0

## Architecture

### Source Independence

Each of the four network sub-sources is a fully independent `EntropySource` implementation. They are registered separately with the `SourceRegistry`, not wrapped in a single composite source. This means:

- Each has its own channel and rate limit
- Each can be started/stopped independently
- Each reports its own metadata and status
- The CLI can enable/disable individual network sources via config

`NetworkConfig` is a convenience for configuration parsing; it does not represent a runtime entity.

### DnsEntropySource

The DNS source spawns a tokio task that loops:

1. For each target hostname in `config.targets`, call `tokio::net::lookup_host()`.
2. Measure resolution time.
3. Hash the query string (hostname) using `std::hash::DefaultHasher` to produce a `u64`.
4. Map the hash to grid coordinates via `hash_to_coordinates()`.
5. Map the resolution time to cell age (fast resolution = low age, slow = high age, capped at 100).
6. Emit `SetCells` events with the computed coordinates.
7. Sleep for `config.interval`.

The hash is deterministic -- the same hostname always maps to the same grid position. This creates stable "landmarks" on the grid that pulse when DNS resolution is active, with their brightness (cell age) reflecting latency.

### PingEntropySource

The ping source spawns a tokio task:

1. For each target in `config.targets`, send an ICMP echo request via `surge-ping`.
2. Await the response with `config.timeout`.
3. On success: compute `latency_to_cell_count()`. Hash the target IP to get a center coordinate. Emit `SetCells` with `cell_count` random coordinates clustered around the center (using the latency as a seed for a deterministic scatter pattern).
4. On timeout: emit an `EnergyPulse` at the target's hashed position with high intensity (network loss = chaos).
5. On unreachable: skip this target, continue to next. Log at Debug level.
6. Sleep for `config.interval`.

The latency mapping uses a baseline of 20ms. Latency at or below baseline produces 1 cell. Each additional 10ms of latency adds 1 cell, capped at 50 cells per ping. This satisfies AC-2 (higher latency = more births).

`surge-ping` requires `CAP_NET_RAW` on Linux or running as root. If the initial ping fails with a permission error, the source sets `SourceStatus::Error` with a message suggesting `setcap cap_net_raw+ep` on the binary. This satisfies AC-5 and AC-6.

### SocketEntropySource

The socket source binds a UDP socket and spawns a tokio task:

1. `tokio::net::UdpSocket::bind(config.bind_addr)`.
2. Loop: `recv_from()` up to `max_packet_size` bytes.
3. Hash the received payload via `hash_to_coordinates()`.
4. Emit `SetCells` with the hashed coordinates.
5. Respect the `CancellationToken` for shutdown.

This is a passive source -- it produces entropy only when something sends data to it. Users can pipe arbitrary data (`echo "entropy" | nc -u 127.0.0.1 19420`) or configure other tools to send UDP telemetry to the socket.

### InterfaceStatsSource

The interface stats source reads Linux sysfs counters:

1. On first poll, read `/sys/class/net/{iface}/statistics/{rx_bytes,tx_bytes,rx_packets,tx_packets}` for each configured interface (or all non-loopback interfaces if none specified).
2. Store as `IfaceSnapshot`.
3. On subsequent polls, compute deltas: `(current - prev) / elapsed_seconds`.
4. Normalize the delta to an entropy density (0.0 to 1.0) using a configurable baseline of 1 MB/s for bytes and 1000 pps for packets.
5. If density > 0.1, emit `SetCells` events. The number of cells is proportional to density. Coordinates are derived from a hash of the interface name plus the current timestamp.
6. Store current snapshot as `prev_stats`.

This source is Linux-specific. On non-Linux platforms, `start()` returns `Err` with a message indicating the platform is unsupported. The source is still compiled (no `#[cfg(target_os)]` on the type itself), but fails gracefully at runtime.

### Coordinate Hashing

`hash_to_coordinates()` is shared across all network sources:

```rust
fn hash_to_coordinates(data: &[u8], grid_extents: &[usize]) -> Vec<usize> {
    let mut hasher = DefaultHasher::new();
    data.hash(&mut hasher);
    let hash = hasher.finish();
    grid_extents.iter().enumerate().map(|(i, &extent)| {
        // Use different bits of the hash for each dimension
        let shift = (i * 16) % 64;
        ((hash >> shift) as usize) % extent
    }).collect()
}
```

For grids with more than 4 dimensions, the hash bits wrap around. This is acceptable because network sources are primarily meaningful in 2D visualizations.

### Feature Gate

All network code is behind `#[cfg(feature = "network")]`:

```toml
[features]
network = ["dep:surge-ping"]

[dependencies]
surge-ping = { version = "0.8", optional = true }
```

`tokio` is already a required dependency of `golf-entropy` for the async runtime. DNS resolution and UDP sockets use `tokio::net` which requires no additional dependencies.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| `surge-ping` requires `CAP_NET_RAW` or root | Ping source unusable for unprivileged users | High | Graceful degradation with a clear error message. Document `setcap` in README. Other network sources work without elevated privileges. |
| `/sys/class/net/` does not exist on non-Linux | `InterfaceStatsSource` fails | Certain on macOS | Runtime error with clear message. Not a compile error -- the type exists but `start()` fails. |
| DNS resolution via `lookup_host` uses the system resolver, not raw DNS | Cannot observe DNS queries made by other processes | Medium | This is intentional for v1 -- we resolve our own targets. Observing system-wide DNS would require `pcap` or eBPF, which is out of scope. |
| UDP socket on a fixed port may conflict | `bind()` fails if port in use | Low | Port is configurable. Default port 19420 is unlikely to conflict. Error message includes the port number. |
| High-frequency polling (100ms ping interval) generates significant ICMP traffic | Network noise, possible rate limiting by upstream | Low | 100ms is the minimum, 1s is the default. Document that aggressive intervals may trigger rate limiting. |

## ADR-007-1: Four Independent Sources over One Composite

**Status:** Accepted

**Context:** The four network entropy mechanisms (DNS, ping, socket, interface stats) could be implemented as one `NetworkEntropySource` with internal multiplexing, or as four separate `EntropySource` implementations.

**Decision:** Four independent sources, each registered separately with the `SourceRegistry`.

**Consequences:**
- Positive: Each source has independent lifecycle, rate limiting, and error handling. Users can enable only the sources they want. A permission error on ping does not affect DNS or socket sources.
- Negative: Four sources consume four channel slots in the registry. Configuration is slightly more verbose (four config blocks instead of one).
- Mitigation: `NetworkConfig` provides a single configuration entry point that fans out to individual source configs. Channel overhead is negligible.

## ADR-007-2: System Resolver over Raw DNS Capture

**Status:** Accepted

**Context:** DNS entropy could come from (a) resolving our own queries via the system resolver, or (b) capturing DNS traffic from other processes via packet capture (`pcap`/`eBPF`).

**Decision:** Use the system resolver (`tokio::net::lookup_host`) to resolve a configurable list of targets.

**Consequences:**
- Positive: No elevated privileges required. No dependency on `libpcap`. Portable across platforms. Deterministic -- we control the query targets.
- Negative: We only see our own DNS activity, not system-wide. The entropy is driven by our polling schedule, not by organic network activity.
- Mitigation: The interface stats source and socket source provide organic, system-driven entropy. DNS is the "stable landmark" source, not the high-entropy source. A future spec could add a `pcap`-based source for true passive DNS observation.
