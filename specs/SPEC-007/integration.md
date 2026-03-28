# SPEC-007: Network Entropy -- Integration

## Consumed Interfaces

### SPEC-003 (Entropy Trait)
- Consumes: `EntropySource` trait, `EntropyEvent` enum, `SourceStatus`, `SourceMetadata`, `SourceRegistry`
- Version: SPEC-003 as of main branch at implementation time
- Contract: All four network sources (`DnsEntropySource`, `PingEntropySource`, `SocketEntropySource`, `InterfaceStatsSource`) implement `EntropySource`. Events emitted are `SetCells(Vec<Vec<usize>>)` and `EnergyPulse { center, radius, intensity }` (ping timeout only).

### SPEC-001 (Core Engine)
- Consumes: Grid extents for coordinate hashing
- Version: SPEC-001 as of main branch at implementation time
- Contract: `hash_to_coordinates()` requires the grid's `extents` (as `Vec<usize>`) to produce valid coordinates. Extents are read at source construction and cached. Grid resizing requires source reconstruction.

## Provided Interfaces

### To SPEC-003 (Entropy Trait) / SourceRegistry
- Provides: `DnsEntropySource`, `PingEntropySource`, `SocketEntropySource`, `InterfaceStatsSource` as registerable `Box<dyn EntropySource>` instances
- Registration: The CLI or config layer constructs configs, creates sources, and registers each independently with the `SourceRegistry`.

### To SPEC-010 (CLI & Config)
- Provides: `NetworkConfig`, `DnsConfig`, `PingConfig`, `SocketConfig`, `IfaceConfig` for CLI flag and TOML config deserialization
- Contract: All config structs derive `serde::Deserialize`. The CLI exposes flags like `--ping-targets`, `--ping-interval`, `--dns-targets`, `--udp-bind`, `--iface-poll`.

## Integration Points

### Shared Utility: hash_to_coordinates
All four network sources use `hash_to_coordinates()` from `crates/golf-entropy/src/network/mod.rs`. This function is internal to the `network` module and not part of the public API. It takes arbitrary `&[u8]` and `&[usize]` extents, producing coordinates guaranteed to be within bounds.

### Cancellation
All four sources use `tokio_util::sync::CancellationToken` for graceful shutdown. When `stop()` is called, the token is cancelled, and the spawned task exits at its next cancellation check point (typically at the top of its polling loop). This ensures no lingering tasks after the source is stopped.

### Platform-Specific Behavior

| Source | Linux | macOS | Notes |
|--------|-------|-------|-------|
| DnsEntropySource | Full support | Full support | Uses `tokio::net::lookup_host` |
| PingEntropySource | Requires `CAP_NET_RAW` | Requires root or group `wheel` | `surge-ping` uses raw sockets |
| SocketEntropySource | Full support | Full support | Standard UDP socket |
| InterfaceStatsSource | Full support | Fails at runtime | Reads `/sys/class/net/` (Linux sysfs only) |

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| ICMP permission denied | `surge-ping` returns `PermissionDenied` error | `start()` returns `Err`. Status set to `Error` with message suggesting `setcap`. Other sources unaffected. |
| DNS resolution timeout | `lookup_host` future times out | Skip this resolution cycle. Log at Debug. Retry on next interval. |
| UDP bind fails (port in use) | `UdpSocket::bind` returns `AddrInUse` | `start()` returns `Err`. Status set to `Error` with message including the port number. |
| `/sys/class/net/` not found | `read_iface_stats` returns `NotFound` | `start()` returns `Err`. Status set to `Error("platform unsupported: /sys/class/net/ not found")`. |
| Network interface disappears mid-operation | `read_iface_stats` returns `NotFound` for a specific interface | Remove interface from `prev_stats`. Log at Warn. Continue polling remaining interfaces. |
| All ping targets unreachable | Every ping returns timeout | Source continues operating, emitting `EnergyPulse` events for each timeout (network chaos = visual chaos). |
