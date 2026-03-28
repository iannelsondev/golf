# SPEC-011: MCP Interface -- Design

## Signatures

### Files Created or Modified

- `crates/golf-cli/src/mcp/mod.rs` -- module root, re-exports
- `crates/golf-cli/src/mcp/server.rs` -- `GolfMcpServer` struct, tool registration, JSON-RPC dispatch
- `crates/golf-cli/src/mcp/tools.rs` -- tool handler functions (inject_pattern, set_rules, get_state, get_stats, control)
- `crates/golf-cli/src/mcp/types.rs` -- request/response types for each tool, JSON schemas
- `crates/golf-cli/src/mcp/transport.rs` -- stdio and SSE transport implementations
- `crates/golf-cli/src/serve.rs` -- `golf serve` subcommand handler, headless and TUI+MCP modes

### Types Introduced

```rust
/// MCP server holding shared references to simulation state.
pub struct GolfMcpServer {
    engine: Arc<Mutex<SimulationEngine<N>>>,  // type-erased via EngineHandle
    registry: Arc<Mutex<SourceRegistry>>,
    frame_state: Arc<FrameState>,
    paused: Arc<AtomicBool>,
    config: ResolvedConfig,
}

/// Type-erased handle to a SimulationEngine<N>.
/// Provides dimension-agnostic operations over the engine.
pub enum EngineHandle {
    D2(SimulationEngine<2>),
    D3(SimulationEngine<3>),
    D4(SimulationEngine<4>),
}

// --- Tool request types ---

/// inject_pattern request
#[derive(Debug, Deserialize, JsonSchema)]
pub struct InjectPatternRequest {
    /// Cell coordinates as arrays of N-dimensional positions
    pub cells: Vec<Vec<usize>>,
    /// Cell state to set (default: 1 = alive)
    #[serde(default = "default_cell_state")]
    pub state: u16,
}

/// set_rules request
#[derive(Debug, Deserialize, JsonSchema)]
pub struct SetRulesRequest {
    /// Birth neighbor counts
    pub birth: Vec<u32>,
    /// Survival neighbor counts
    pub survival: Vec<u32>,
    /// Neighborhood type
    #[serde(default = "default_neighborhood")]
    pub neighborhood: String,  // "moore" | "vonneumann"
}

/// get_state request (all fields optional for full-grid query)
#[derive(Debug, Deserialize, JsonSchema)]
pub struct GetStateRequest {
    /// Origin of the region to query
    pub origin: Option<Vec<usize>>,
    /// Size of the region to query
    pub size: Option<Vec<usize>>,
}

/// get_state response
#[derive(Debug, Serialize)]
pub struct GetStateResponse {
    pub generation: u64,
    pub dimensions: Vec<usize>,
    pub cells: Vec<CellEntry>,
    pub total_cells_in_region: usize,
    pub live_cells_in_region: usize,
}

/// A single live cell in the response (dead cells omitted for compactness).
#[derive(Debug, Serialize)]
pub struct CellEntry {
    pub coord: Vec<usize>,
    pub age: u16,
}

/// get_stats response
#[derive(Debug, Serialize)]
pub struct GetStatsResponse {
    pub generation: u64,
    pub live_cells: u64,
    pub total_cells: u64,
    pub dimensions: Vec<usize>,
    pub rules: RulesDescription,
    pub active_sources: Vec<String>,
    pub paused: bool,
    pub fps: f64,
}

#[derive(Debug, Serialize)]
pub struct RulesDescription {
    pub birth: Vec<u32>,
    pub survival: Vec<u32>,
    pub neighborhood: String,
}

/// control request
#[derive(Debug, Deserialize, JsonSchema)]
pub struct ControlRequest {
    /// Action to perform
    pub action: ControlAction,
}

#[derive(Debug, Deserialize, JsonSchema)]
pub enum ControlAction {
    #[serde(rename = "pause")]
    Pause,
    #[serde(rename = "resume")]
    Resume,
    #[serde(rename = "step")]
    Step,
    #[serde(rename = "reset")]
    Reset,
}

/// MCP transport configuration.
pub enum TransportConfig {
    Stdio,
    Sse {
        bind: SocketAddr,
        token: Option<String>,
    },
}
```

### Functions / Contracts

```rust
// --- EngineHandle ---
impl EngineHandle {
    /// Construct from resolved config (dispatches on dimensions).
    pub fn new(config: &ResolvedConfig) -> Result<Self>;

    /// Inject cells at the given coordinates. Validates coordinate dimensionality.
    pub fn inject_cells(&mut self, cells: &[Vec<usize>], state: u16) -> Result<()>;

    /// Replace the active rule set.
    pub fn set_rules(&mut self, birth: &[u32], survival: &[u32], neighborhood: &str) -> Result<()>;

    /// Read cell states in a region. Returns only live cells for compactness.
    pub fn get_state(&self, origin: Option<&[usize]>, size: Option<&[usize]>) -> Result<GetStateResponse>;

    /// Read simulation statistics.
    pub fn get_stats(&self, active_sources: &[String], paused: bool, fps: f64) -> GetStatsResponse;

    /// Advance one generation.
    pub fn step(&mut self);

    /// Reset: clear the grid, reset generation counter.
    pub fn reset(&mut self);

    /// Grid extents.
    pub fn extents(&self) -> Vec<usize>;

    /// Current generation.
    pub fn generation(&self) -> u64;
}

// --- GolfMcpServer ---
impl GolfMcpServer {
    /// Construct the MCP server with shared state.
    pub fn new(
        engine: Arc<Mutex<EngineHandle>>,
        registry: Arc<Mutex<SourceRegistry>>,
        frame_state: Arc<FrameState>,
        config: ResolvedConfig,
    ) -> Self;

    /// Return the MCP tools manifest (tool name, description, JSON schema for each tool).
    pub fn tools_list(&self) -> Vec<ToolDefinition>;

    /// Dispatch a tool call by name to the appropriate handler.
    pub async fn call_tool(&self, name: &str, arguments: Value) -> Result<Value>;
}

// --- Tool handlers (called by call_tool dispatch) ---

/// Inject a pattern into the simulation grid.
fn handle_inject_pattern(engine: &mut EngineHandle, req: InjectPatternRequest) -> Result<Value>;

/// Change the active rule set.
fn handle_set_rules(engine: &mut EngineHandle, req: SetRulesRequest) -> Result<Value>;

/// Query grid state for a region.
fn handle_get_state(engine: &EngineHandle, req: GetStateRequest) -> Result<Value>;

/// Query simulation statistics.
fn handle_get_stats(server: &GolfMcpServer) -> Result<Value>;

/// Simulation lifecycle control (pause, resume, step, reset).
fn handle_control(server: &GolfMcpServer, req: ControlRequest) -> Result<Value>;

// --- Transport ---

/// Run MCP server over stdio (JSON-RPC on stdin/stdout).
pub async fn run_stdio_transport(server: Arc<GolfMcpServer>) -> Result<()>;

/// Run MCP server over SSE (HTTP with Server-Sent Events).
pub async fn run_sse_transport(
    server: Arc<GolfMcpServer>,
    bind: SocketAddr,
    token: Option<String>,
) -> Result<()>;

// --- Serve subcommand ---

/// Entry point for `golf serve`. Constructs engine, registry, optional TUI,
/// MCP server, and runs the appropriate transport.
pub async fn handle_serve(args: ServeArgs) -> Result<()>;
```

## Architecture

### MCP Protocol Compliance

The server implements the Model Context Protocol over JSON-RPC 2.0. On initialization, the client sends `initialize` and the server responds with capabilities including the tools list. Tool calls arrive as `tools/call` requests with a tool name and JSON arguments.

The five tools and their MCP schema descriptions:

| Tool | Description | Input Schema |
|------|-------------|-------------|
| `inject_pattern` | Inject live cells at N-dimensional coordinates | `{ cells: [[x,y,...]], state?: u16 }` |
| `set_rules` | Change birth/survival rules and neighborhood type | `{ birth: [u32], survival: [u32], neighborhood?: "moore"\|"vonneumann" }` |
| `get_state` | Query cell states in a grid region | `{ origin?: [x,y,...], size?: [w,h,...] }` |
| `get_stats` | Get simulation statistics | `{}` (no parameters) |
| `control` | Pause, resume, step, or reset the simulation | `{ action: "pause"\|"resume"\|"step"\|"reset" }` |

### EngineHandle: Type-Erasing const N

The core challenge is that `SimulationEngine<const N: usize>` requires a compile-time dimension, but the MCP server receives coordinates as variable-length JSON arrays. `EngineHandle` is an enum that wraps `SimulationEngine<2>`, `SimulationEngine<3>`, and `SimulationEngine<4>`, providing dimension-agnostic methods that validate coordinate lengths at runtime.

```
MCP tool call (JSON)
    |
    v
Deserialize to typed request (InjectPatternRequest)
    |
    v
EngineHandle::inject_cells()
    |  match self { D2(e) => ..., D3(e) => ..., D4(e) => ... }
    |  validate coordinate length == N
    |  convert Vec<usize> to [usize; N]
    v
SimulationEngine<N>::inject_events()
```

This is the same dispatch pattern as SPEC-010's dimension dispatch, centralized in one enum.

### Shared State Model

When the MCP server runs alongside a simulation loop (headless or TUI), state is shared through `Arc<Mutex<_>>`:

```
                  Arc<Mutex<EngineHandle>>
                 /          |             \
Simulation loop    MCP server    (TUI renderer reads FrameState)
  step()            call_tool()    render_frame()
  inject_events()   get_state()
                    set_rules()
```

The `Mutex<EngineHandle>` is held briefly for each operation:
- Simulation step: lock, drain, inject, step, unlock, publish to FrameState
- MCP get_state: lock, read grid, unlock, serialize response
- MCP inject_pattern: lock, inject cells, unlock
- MCP control("step"): lock, step, unlock

Lock contention is minimal because MCP requests are infrequent relative to the simulation tick rate. The 50ms response time NFR is easily met -- a lock acquisition plus grid read for a 200x50 region is sub-millisecond.

### Headless Mode

When `golf serve` runs without `--tui`, there is no renderer and no crossterm event loop. The simulation runs in a tokio task that ticks at the configured FPS. MCP commands are the only way to interact.

```
Headless mode:
  tokio task 1: simulation tick loop (if not paused)
  tokio task 2: MCP transport (stdio or SSE)
  shared: Arc<Mutex<EngineHandle>>, Arc<AtomicBool> (paused)

TUI mode (--tui):
  tokio task 1: simulation tick loop
  tokio task 2: MCP transport
  std thread 1: crossterm input loop
  std thread 2: renderer loop
  shared: above + Arc<FrameState>
```

In headless mode, the simulation starts paused by default. The MCP client controls when to step or resume. This is the expected mode for LLM agents that want precise control over the simulation.

### Stdio Transport

The default transport reads JSON-RPC messages from stdin and writes responses to stdout. This is the standard MCP transport for Claude Code and similar tools.

```
stdin  --> line reader --> JSON-RPC parser --> dispatch --> GolfMcpServer::call_tool()
stdout <-- JSON-RPC serializer <-- Result<Value> <---------/
```

When `--tui` is active alongside stdio transport, the TUI renders to the alternate screen buffer. Stdout for JSON-RPC must not interfere with TUI output. This is achieved by:
1. The TUI uses crossterm's alternate screen (writes to the terminal directly, not stdout).
2. JSON-RPC responses are written to stdout, which the MCP client reads.
3. The MCP client is connected to the process's stdin/stdout pipes, not the terminal.

This works because MCP clients (Claude Code, etc.) launch `golf serve` as a subprocess and pipe stdin/stdout. The TUI writes to `/dev/tty` directly when `--tui` is active, bypassing the piped stdout.

### SSE Transport

The optional SSE transport runs an HTTP server (using `axum`) that accepts:
- `GET /sse` -- Server-Sent Events stream for server-to-client messages
- `POST /message` -- client-to-server JSON-RPC requests

When `--token` is provided, all requests must include `Authorization: Bearer <token>`. Requests without a valid token receive 401 Unauthorized.

SSE transport is secondary and may be deferred to a later iteration. Stdio is the priority.

### get_state Response Format

For grids up to 200x50 (10,000 cells), `get_state` returns only live cells to keep response size manageable. A full grid of 10,000 cells where 30% are alive produces about 3,000 `CellEntry` objects, which serializes to roughly 100KB of JSON.

For larger grids or full-grid queries, the response includes a `total_cells_in_region` count so the client knows the grid size without receiving every dead cell.

If no region is specified and the grid exceeds 100,000 total cells, the server returns an error suggesting a bounded region query.

### Simulation Control Semantics

| Action | Behavior |
|--------|----------|
| `pause` | Set paused flag. Simulation loop skips tick. MCP queries still work. |
| `resume` | Clear paused flag. Simulation loop resumes ticking. |
| `step` | If paused: advance exactly one generation. If running: pause first, then step. |
| `reset` | Clear all cells to dead. Reset generation to 0. Keep rules and sources. Pause. |

## Data Model

```
GolfMcpServer
  engine: Arc<Mutex<EngineHandle>>
  registry: Arc<Mutex<SourceRegistry>>
  frame_state: Arc<FrameState>
  paused: Arc<AtomicBool>
  config: ResolvedConfig

EngineHandle
  D2(SimulationEngine<2>)
  | D3(SimulationEngine<3>)
  | D4(SimulationEngine<4>)

TransportConfig
  Stdio
  | Sse { bind: SocketAddr, token: Option<String> }
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Mutex contention between sim loop and MCP tool calls under high MCP request rate | Simulation stutters, MCP responses slow | Low -- MCP requests are human/LLM-initiated, not high-frequency | Mutex hold times are microseconds. If contention appears, switch to `tokio::sync::RwLock` (read-heavy workload). |
| get_state on large grids produces huge JSON responses | Slow serialization, memory pressure, client confusion | Medium | Cap full-grid queries at 100K cells. Require region bounds for larger grids. Return only live cells. |
| Stdio transport and TUI both writing to stdout | Garbled output, broken JSON-RPC | Medium | TUI writes to `/dev/tty` directly. JSON-RPC uses process stdout. Verify separation in integration tests. |
| rmcp crate API changes or is too immature | Need to rewrite transport layer | Medium | Abstract transport behind a trait. If rmcp is unusable, implement JSON-RPC manually (the protocol is simple: read line, parse JSON, dispatch, write response). |
| Coordinate dimension mismatch in inject_pattern | Runtime error instead of clear feedback | Medium | EngineHandle validates coordinate length and returns a descriptive error: "expected 2-dimensional coordinates, got 3". |
| SSE transport without TLS exposes simulation control on the network | Unauthorized access to simulation | Low -- SSE is local-use, optional | Bind to 127.0.0.1 by default. Bearer token auth. Document that SSE without a token is insecure. |

## ADR-001: EngineHandle Enum over Trait Object

**Status:** Accepted

**Context:** The MCP server needs to operate on a `SimulationEngine<N>` without knowing N at compile time. Two options: (a) an enum wrapping each supported N, (b) a trait object erasing N.

**Decision:** Use `EngineHandle` enum with variants `D2`, `D3`, `D4`.

**Consequences:**
- No vtable overhead. Each method is a match with three arms.
- Adding a new dimension requires adding an enum variant and match arms -- mechanical but explicit.
- Consistent with SPEC-010's dimension dispatch pattern. The EngineHandle can be shared between CLI and MCP code paths.
- A trait object would require defining a dimension-agnostic trait with `Vec<usize>` coordinates on every method, duplicating the SPEC-001 interface. The enum avoids this by converting `Vec<usize>` to `[usize; N]` at the match boundary.

## ADR-002: Live-Cells-Only Response over Full Grid Serialization

**Status:** Accepted

**Context:** `get_state` could return the full grid (every cell including dead) or only live cells. Game of Life grids are typically 10-30% alive.

**Decision:** Return only live cells as `Vec<CellEntry>` with coordinates and age.

**Consequences:**
- Response size is proportional to live cell count, not grid volume. A 200x50 grid at 20% density produces ~2,000 entries instead of 10,000.
- Clients that need the full grid layout can reconstruct it from the extents (returned in the response) and the live cell list.
- Dead cells have no interesting data (age=0), so omitting them loses nothing.
- Slightly more work for clients that want a dense 2D array -- they must scatter live cells into a grid. This is a reasonable trade-off for a tool-calling LLM that primarily wants to understand patterns, not render pixels.

## ADR-003: Stdio-First over HTTP-First Transport

**Status:** Accepted

**Context:** MCP supports stdio and SSE transports. The primary use case is Claude Code launching `golf serve` as a subprocess.

**Decision:** Stdio is the default and required transport. SSE is optional and may be deferred.

**Consequences:**
- Stdio requires zero network configuration. No port binding, no auth for local use.
- Claude Code, VS Code MCP extensions, and most MCP clients expect stdio.
- SSE enables remote or multi-client access but adds HTTP server complexity (axum dependency, auth, CORS).
- Implementing stdio first keeps the initial scope small. SSE can be added behind a feature flag without changing the server core.
