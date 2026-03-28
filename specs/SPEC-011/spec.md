# SPEC-011: MCP Interface

**Status:** approved
**Spec Type:** api
**Feature Area:** MCP Interface
**Dependencies:** SPEC-001, SPEC-003, SPEC-010
**Issue:** #11

## Problem Statement

Golf should be controllable by LLM agents, editors, and other tools via the Model Context Protocol (MCP). An MCP server embedded in the golf binary exposes simulation control, entropy injection, state queries, and rule mutation as tools. This lets Claude Code, VS Code extensions, or custom agents interact with a running simulation programmatically — creating patterns, evolving rules, and querying state without the TUI.

## Functional Requirements

- **FR-1:** `golf serve` subcommand — start golf as an MCP server over stdio (default) or SSE transport, with simulation running headless or with TUI
- **FR-2:** MCP tool: `inject_pattern(cells: [[x,y,...]], dimensions: N)` — inject cells at N-dimensional coordinates into the running simulation
- **FR-3:** MCP tool: `set_rules(birth: [u32], survival: [u32], neighborhood: "moore"|"vonneumann")` — change the active ruleset at runtime
- **FR-4:** MCP tool: `get_state(region?: {origin: [x,y,...], size: [w,h,...]})` — return current grid state for a region (or full grid if small enough)
- **FR-5:** MCP tool: `get_stats()` — return generation count, live cell count, dimensions, active sources, current rules, FPS
- **FR-6:** MCP tool: `control(action: "pause"|"resume"|"step"|"reset")` — simulation lifecycle control

## Non-Functional Requirements

- **NFR-1:** MCP responses must complete in under 50ms for state queries on grids up to 200x50
- **NFR-2:** MCP server must not interfere with TUI rendering when both are active

## Acceptance Criteria

- **AC-1:** Given `golf serve --transport stdio`, when an MCP client connects, then the tools list is returned with all 5 tools
- **AC-2:** Given `inject_pattern` called with a glider's coordinates, when the simulation steps, then the glider is visible and behaves correctly
- **AC-3:** Given `set_rules` called with B36/S23, when the simulation continues, then the new rules are applied to subsequent steps
- **AC-4:** Given `get_state` called with a 10x10 region, when the simulation is running, then the current cell states for that region are returned as JSON
- **AC-5:** Given `control("pause")` followed by `control("step")`, when called, then the simulation pauses and advances exactly one generation
- **AC-6:** Given `golf serve` with TUI enabled, when MCP commands arrive, then both the TUI and MCP responses reflect the same simulation state

## Constraints

- MCP implementation follows the MCP specification — standard tool definitions, JSON-RPC over stdio/SSE
- No authentication required for local stdio transport; SSE transport should support optional bearer token
