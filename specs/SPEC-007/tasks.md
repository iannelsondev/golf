# SPEC-007: Network Entropy -- Tasks

## Task List

### T1: Shared utilities and config types
- **Scope:** Create `crates/golf-entropy/src/network/config.rs` and `crates/golf-entropy/src/network/mod.rs`. Define `NetworkConfig`, `DnsConfig`, `PingConfig`, `SocketConfig`, `IfaceConfig`. Implement `hash_to_coordinates()` and `latency_to_cell_count()`. Add `surge-ping` as optional dependency behind `network` feature.
- **AC coverage:** Foundation for all ACs
- **Estimate:** S

### T2: DnsEntropySource
- **Scope:** Create `crates/golf-entropy/src/network/dns.rs`. Implement the DNS resolution loop using `tokio::net::lookup_host`. Hash query strings to coordinates. Map resolution time to cell age. Implement `EntropySource` trait. Cancellation via `CancellationToken`.
- **AC coverage:** AC-1 (DNS queries produce coordinates on grid)
- **Estimate:** S

### T3: PingEntropySource
- **Scope:** Create `crates/golf-entropy/src/network/ping.rs`. Implement the ping loop using `surge-ping`. Map latency to cell count. Hash target IP to center coordinate. Emit `EnergyPulse` on timeout. Handle permission errors gracefully.
- **AC coverage:** AC-2 (higher latency = more births), AC-6 (unreachable target skipped gracefully)
- **Estimate:** M

### T4: SocketEntropySource
- **Scope:** Create `crates/golf-entropy/src/network/socket.rs`. Bind UDP socket, receive packets, hash payload to coordinates, emit `SetCells`. Cancellation via `CancellationToken`.
- **AC coverage:** AC-3 (incoming UDP packets hashed to coordinates)
- **Estimate:** S

### T5: InterfaceStatsSource
- **Scope:** Create `crates/golf-entropy/src/network/iface.rs`. Implement `read_iface_stats()` for `/sys/class/net/*/statistics/`. Track previous snapshots, compute deltas, normalize to entropy density. Emit `SetCells` proportional to throughput. Handle missing sysfs gracefully.
- **AC coverage:** AC-4 (high throughput = increased entropy density)
- **Estimate:** M

### T6: Unit tests
- **Scope:** Test `hash_to_coordinates()` determinism and bounds. Test `latency_to_cell_count()` scaling. Test `delta_to_entropy_density()` normalization. Test graceful degradation paths for each source (mock permission errors, missing sysfs). Test `DnsEntropySource` with localhost resolution.
- **AC coverage:** AC-1 through AC-6 (unit-level verification)
- **Estimate:** M

### T7: Integration tests
- **Scope:** Integration test that starts `DnsEntropySource` resolving `localhost`, verifies events are produced. Integration test for `SocketEntropySource` -- bind, send a UDP packet from the test, verify `SetCells` event received. Gate ping and iface tests behind environment variables (`GOLF_TEST_PING=1`, `GOLF_TEST_IFACE=1`) since they require privileges or Linux.
- **AC coverage:** AC-1, AC-3, AC-5 (end-to-end verification)
- **Estimate:** S

## Dependency Order

```
T1 (shared utils) --> T2 (DNS)
                  --> T3 (ping)
                  --> T4 (socket)
                  --> T5 (iface stats)
                      all of above --> T6 (unit tests) --> T7 (integration tests)
```

T2, T3, T4, and T5 are independent of each other (all depend only on T1). They can be implemented in parallel.
