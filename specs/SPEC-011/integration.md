# SPEC-011: MCP Interface -- Integration

## Consumed Interfaces

### SPEC-001 (Core Engine)
- **Version:** SPEC-001 as of current main branch at implementation time
- **Consumes:**
  - `SimulationEngine<const N: usize>` -- wrapped inside `EngineHandle` enum for type erasure
  - `SimulationEngine::inject_events(events)` -- used by `inject_pattern` tool to set cells
  - `SimulationEngine::step()` -- used by `control("step")` tool
  - `SimulationEngine::grid() -> &NdGrid<N>` -- used by `get_state` to read cell values
  - `SimulationEngine::generation() -> u64` -- used by `get_stats` response
  - `SimulationEngine::rules() -> &RuleSet` -- used by `get_stats` to describe current rules
  - `SimulationEngine::set_rules(rules)` -- used by `set_rules` tool
  - `NdGrid<N>::get(coord) -> Cell` -- cell reads for region queries
  - `NdGrid<N>::extents() -> [usize; N]` -- grid dimensions for response metadata
  - `NdGrid<N>::len() -> usize` -- total cell count for stats
  - `RuleSet` struct -- constructed from `set_rules` tool parameters
  - `NeighborhoodKind` enum -- parsed from the `neighborhood` string parameter
  - `EntropyEvent<N>::SetCells` -- constructed from `inject_pattern` coordinates
  - `Cell` type alias (`u16`) -- read from grid, returned as `age` in CellEntry
- **Contract:** SimulationEngine must be safe to access from multiple threads when wrapped in `Arc<Mutex<>>`. All methods must be callable while the simulation is paused. `inject_events` must accept events at any point between steps.

### SPEC-003 (Entropy Trait)
- **Version:** SPEC-003 as of current main branch at implementation time
- **Consumes:**
  - `SourceRegistry::list() -> Vec<(SourceId, SourceMetadata)>` -- used by `get_stats` to report active sources
  - `SourceMetadata::name` -- source names included in stats response
  - `SourceRegistry::drain(budget)` -- called by simulation tick loop (not directly by MCP, but the registry is shared state)
- **Contract:** SourceRegistry must be safe to access from multiple threads when wrapped in `Arc<Mutex<>>`. `list()` must be callable concurrently with `drain()`.

### SPEC-010 (CLI & Config)
- **Version:** SPEC-010 as of current main branch at implementation time
- **Consumes:**
  - `ServeArgs` struct -- parsed by clap, passed to `handle_serve()`
  - `Command::Serve` variant -- the MCP serve subcommand lives in the SPEC-010 CLI enum
  - `GolfConfig` / `ResolvedConfig` -- MCP server uses the same config pipeline to configure the initial simulation state
  - `GolfConfig::load_default()` -- base config for the serve session
  - `ResolvedConfig` -- used to construct EngineHandle, SourceRegistry, optional Renderer
  - Config resolution pipeline (load -> merge -> resolve) -- reused as-is
- **Contract:** The `Serve` variant must exist in the `Command` enum. `ServeArgs` must contain `transport`, `tui`, `bind`, and `token` fields. The config resolution pipeline must be callable independently of the `Run` subcommand path.

## Provided Interfaces

### To MCP Clients (Claude Code, VS Code, custom agents)
- **Provides:**
  - MCP tools manifest via `initialize` response: `inject_pattern`, `set_rules`, `get_state`, `get_stats`, `control`
  - JSON-RPC 2.0 over stdio (default) or SSE (optional)
  - Each tool has a JSON Schema describing its input parameters
  - Responses are JSON objects with typed fields (no arbitrary structure)
- **Contract:** The server follows MCP specification for tool listing, tool calling, and error reporting. Tool names and parameter schemas are stable within a major version.

## Integration Patterns

### Serve Subcommand Entry Point

```
golf serve [--transport stdio|sse] [--tui] [--bind addr] [--token tok]
    |
    v
SPEC-010 CLI parsing (clap)
    |
    v
handle_serve(ServeArgs)
    |
    v
1. Load and resolve GolfConfig (same pipeline as `golf run`)
2. Construct EngineHandle from ResolvedConfig
3. Construct SourceRegistry, register default sources
4. Wrap in Arc<Mutex<>>
5. If --tui: construct Renderer, FrameState, spawn render thread + input thread
6. Construct GolfMcpServer with shared state
7. Spawn simulation tick loop (tokio task)
8. Match transport:
   - stdio: run_stdio_transport(server)
   - sse: run_sse_transport(server, bind, token)
9. Await shutdown signal (client disconnect or SIGINT)
10. Cleanup: stop sources, restore terminal (if TUI)
```

### Shared State Topology

```
Arc<Mutex<EngineHandle>> ----+---- Simulation tick loop (step, inject)
                             |
                             +---- GolfMcpServer (inject_pattern, set_rules, get_state, control)

Arc<Mutex<SourceRegistry>> --+---- Simulation tick loop (drain)
                             |
                             +---- GolfMcpServer (get_stats: list sources)

Arc<FrameState> -------------+---- Simulation tick loop (publish snapshot)
                             |
                             +---- Renderer (read snapshot) [only with --tui]

Arc<AtomicBool> (paused) ----+---- Simulation tick loop (skip tick if paused)
                             |
                             +---- GolfMcpServer (control: pause/resume)
                             |
                             +---- Input loop (space key: toggle) [only with --tui]
```

### MCP + TUI Coexistence

When `--tui` is active:
- The renderer reads `FrameState` (lock-free via Arc clone)
- The MCP server mutates state through `Arc<Mutex<EngineHandle>>`
- Both the TUI input loop and MCP `control` tool can pause/resume via the shared `AtomicBool`
- MCP `inject_pattern` and `set_rules` take effect on the next simulation tick, which the TUI renders

No special synchronization is needed beyond the existing `Mutex<EngineHandle>` because:
- The simulation tick loop is the only writer to the grid (it holds the lock during drain+inject+step)
- MCP tool calls that mutate state (inject, set_rules, control) acquire the same lock
- The renderer only reads `FrameState`, which is updated atomically by the tick loop

### Error Handling at MCP Boundary

MCP tool calls map internal errors to JSON-RPC error responses:

| Internal Error | JSON-RPC Code | Message |
|---------------|---------------|---------|
| Coordinate dimension mismatch | -32602 (Invalid params) | "expected 2-dimensional coordinates, got 3" |
| Coordinate out of bounds | -32602 (Invalid params) | "coordinate [201, 50] exceeds grid extents [200, 50]" |
| Unknown neighborhood type | -32602 (Invalid params) | "unknown neighborhood 'foo', expected 'moore' or 'vonneumann'" |
| Region too large (>100K cells) | -32602 (Invalid params) | "requested region contains 500000 cells, max 100000; specify a smaller region" |
| Internal engine error | -32603 (Internal error) | "simulation engine error: <context>" |
