# SPEC-011: MCP Interface -- Tasks

## Task 1: EngineHandle Type Erasure
**AC coverage:** AC-2, AC-3, AC-4, AC-5
**Files:** `crates/golf-cli/src/mcp/mod.rs`, `crates/golf-cli/src/mcp/types.rs`

1. Define `EngineHandle` enum with variants `D2(SimulationEngine<2>)`, `D3(SimulationEngine<3>)`, `D4(SimulationEngine<4>)`
2. Implement `EngineHandle::new(config: &ResolvedConfig) -> Result<Self>` -- match on dimensions, construct engine with extents, rules, edges
3. Implement `inject_cells(&mut self, cells: &[Vec<usize>], state: u16) -> Result<()>`:
   - Match on variant to get concrete `SimulationEngine<N>`
   - Validate each coordinate has length N
   - Convert `Vec<usize>` to `[usize; N]` (try_into, map error to descriptive message)
   - Build `EntropyEvent::SetCells` and call `engine.inject_events()`
4. Implement `set_rules(&mut self, birth, survival, neighborhood) -> Result<()>`:
   - Parse neighborhood string to `NeighborhoodKind`
   - Construct `RuleSet`, call `engine.set_rules()`
5. Implement `get_state(&self, origin, size) -> Result<GetStateResponse>`:
   - If no region specified and grid > 100K cells, return error
   - Iterate region, collect live cells as `CellEntry` (coord, age)
   - Return `GetStateResponse` with generation, dimensions, cells, counts
6. Implement `get_stats`, `step`, `reset`, `extents`, `generation`
7. Write tests: inject into D2 engine, get_state returns correct cells, dimension mismatch errors, set_rules changes behavior

## Task 2: MCP Tool Request/Response Types
**AC coverage:** AC-1
**Files:** `crates/golf-cli/src/mcp/types.rs`

1. Define `InjectPatternRequest`, `SetRulesRequest`, `GetStateRequest`, `ControlRequest`, `ControlAction` with serde derives
2. Define `GetStateResponse`, `GetStatsResponse`, `RulesDescription`, `CellEntry` with Serialize
3. Implement `JsonSchema` derives or manual schema generation for all request types (for MCP tool manifest)
4. Define `ToolDefinition` struct matching MCP spec: `{ name, description, inputSchema }`
5. Implement `fn tool_definitions() -> Vec<ToolDefinition>` returning all five tools with their JSON schemas
6. Write tests: deserialize example JSON for each request type, serialize example responses, verify JSON schema output

## Task 3: GolfMcpServer and Tool Dispatch
**AC coverage:** AC-1, AC-2, AC-3, AC-4, AC-5
**Files:** `crates/golf-cli/src/mcp/server.rs`, `crates/golf-cli/src/mcp/tools.rs`

1. Define `GolfMcpServer` struct with `Arc<Mutex<EngineHandle>>`, `Arc<Mutex<SourceRegistry>>`, `Arc<FrameState>`, `Arc<AtomicBool>` (paused), `ResolvedConfig`
2. Implement `GolfMcpServer::new(...)` constructor
3. Implement `tools_list(&self) -> Vec<ToolDefinition>` -- return the five tool definitions
4. Implement `call_tool(&self, name: &str, arguments: Value) -> Result<Value>`:
   - Match on tool name
   - Deserialize arguments to the typed request struct
   - Acquire lock on engine/registry as needed
   - Call the appropriate handler
   - Serialize response to `Value`
   - Map errors to JSON-RPC error codes
5. Implement tool handlers:
   - `handle_inject_pattern`: lock engine, call inject_cells, return success with injected count
   - `handle_set_rules`: lock engine, call set_rules, return success with new rule description
   - `handle_get_state`: lock engine, call get_state, return GetStateResponse
   - `handle_get_stats`: lock engine + registry, gather stats, return GetStatsResponse
   - `handle_control`: match action, update paused flag or call step/reset
6. Write tests: call_tool with valid JSON for each tool, call_tool with invalid tool name returns error, call_tool with bad params returns -32602

## Task 4: Stdio Transport
**AC coverage:** AC-1, AC-6
**Files:** `crates/golf-cli/src/mcp/transport.rs`

1. Implement `run_stdio_transport(server: Arc<GolfMcpServer>) -> Result<()>`:
   - Read lines from stdin (tokio BufReader)
   - Parse each line as JSON-RPC 2.0 request
   - Handle `initialize` request: respond with server info and capabilities (tools list)
   - Handle `tools/list` request: return tools_list()
   - Handle `tools/call` request: extract tool name and arguments, call server.call_tool(), wrap result in JSON-RPC response
   - Handle `notifications/initialized`: acknowledge, no response needed
   - Write JSON-RPC response as a single line to stdout (tokio BufWriter, flush after each message)
   - On stdin EOF: return Ok(()) (client disconnected)
2. Implement JSON-RPC 2.0 framing: `{ "jsonrpc": "2.0", "id": N, "method": "...", "params": {...} }`
3. Handle JSON-RPC errors: parse errors (-32700), method not found (-32601), invalid params (-32602)
4. Write tests: send initialize + tools/list + tools/call sequence over in-memory pipe, verify responses

## Task 5: Serve Subcommand Handler
**AC coverage:** AC-1, AC-6
**Files:** `crates/golf-cli/src/serve.rs`

1. Implement `handle_serve(args: ServeArgs) -> Result<()>`:
   - Load and resolve GolfConfig (reuse SPEC-010 config pipeline)
   - Construct EngineHandle from ResolvedConfig
   - Construct SourceRegistry, register and start default sources
   - Wrap engine and registry in Arc<Mutex<>>
   - Create Arc<AtomicBool> for paused state (default: true in headless, false with --tui)
   - If `--tui`: construct Renderer, FrameState, spawn render thread and input loop thread
   - Construct GolfMcpServer
   - Spawn simulation tick loop as tokio task:
     - Loop at target FPS
     - If not paused: lock engine, drain registry, inject events, step, publish to FrameState
   - Match transport:
     - "stdio": call run_stdio_transport(server)
     - "sse": call run_sse_transport(server, bind, token) (if implemented)
   - On shutdown: stop sources, cleanup renderer (if TUI), return
2. Wire into SPEC-010's main.rs: `Command::Serve(args) => handle_serve(args).await`
3. Write integration test: launch serve with stdio, send initialize, verify tools list response

## Task 6: SSE Transport (Optional, Deferrable)
**AC coverage:** none (NFR: alternative transport)
**Files:** `crates/golf-cli/src/mcp/transport.rs`

1. Add `axum` dependency behind a `sse` feature flag
2. Implement `run_sse_transport(server, bind, token) -> Result<()>`:
   - Create axum router with:
     - `GET /sse` -- SSE stream for server-to-client messages
     - `POST /message` -- client-to-server JSON-RPC requests
   - If token is Some: add auth middleware that checks `Authorization: Bearer <token>`
   - Bind to the specified address (default 127.0.0.1:3000)
   - Dispatch incoming POST requests through server.call_tool()
   - Send responses as SSE events
3. Write tests: POST a tools/call request, verify SSE event response, verify 401 without token

## Task 7: TUI + MCP Coexistence
**AC coverage:** AC-6
**Files:** `crates/golf-cli/src/serve.rs`

1. When `--tui` is active alongside MCP:
   - Open `/dev/tty` for TUI rendering instead of stdout (so JSON-RPC owns stdout)
   - Construct crossterm backend with the tty file descriptor
   - Spawn input loop reading from `/dev/tty` instead of stdin (so JSON-RPC owns stdin)
   - Both MCP tool calls and TUI keybinds operate on the same shared state
2. Verify: MCP `inject_pattern` causes cells to appear in TUI on next frame
3. Verify: MCP `control("pause")` stops TUI animation, space key in TUI resumes it
4. Write integration test: start serve with --tui flag, verify TUI opens alternate screen, send MCP inject_pattern, verify no stdout corruption
