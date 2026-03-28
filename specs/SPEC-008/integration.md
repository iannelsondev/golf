# SPEC-008: AI Integration -- Integration

## Consumed Interfaces

### SPEC-003 (Entropy Trait)
- Consumes: `EntropySource` trait, `EntropyEvent` enum (`ApplyPattern`, `MutateRules`, `SetCells`), `SourceStatus`, `SourceMetadata`
- Version: SPEC-003 as of main branch at implementation time
- Contract: `AiEntropySource` implements `EntropySource`. Events emitted are `ApplyPattern { cells, offset }` for pattern generation, `MutateRules(RuleSet)` for rule evolution, and `SetCells(Vec<Vec<usize>>)` for conversation-driven cell placement.

### SPEC-001 (Core Engine)
- Consumes: `NdGrid<N>` (read-only via `grid()` and `cells()`), `RuleSet`, `SimulationEngine::generation()`, grid `extents()`
- Version: SPEC-001 as of main branch at implementation time
- Contract: `build_snapshot()` reads from `NdGrid<N>` to produce a `GridSnapshot`. It accesses `grid.cells()` for the cell buffer, `grid.extents()` for dimensions, and `engine.rules()` for the current rule set. This function runs on the simulation thread and must complete in microseconds for target grid sizes (200x50).

### State Delivery Channel
- The simulation engine (SPEC-001) must publish `GridSnapshot` values via `tokio::sync::watch::Sender<GridSnapshot>`.
- `AiEntropySource` receives a `watch::Receiver<GridSnapshot>` via `set_state_channel()` before `start()` is called.
- If `set_state_channel()` is not called before `start()`, the AI source operates without state context: pattern generation works (user provides the prompt), but evolution and narration are disabled (they require state observation).

## Provided Interfaces

### To SPEC-003 (Entropy Trait) / SourceRegistry
- Provides: `AiEntropySource` as a registerable `Box<dyn EntropySource>`
- Registration: The CLI constructs an `AiConfig`, creates an `OpenAiCompatibleClient`, wraps it in `AiEntropySource`, connects the state channel, and registers with `SourceRegistry`.

### To SPEC-002 (Rendering)
- Provides: Narration text via `AiEntropySource::narration_rx() -> Option<Receiver<String>>`
- Contract: The rendering layer polls this receiver and overlays narration text on the terminal display. If the receiver is `None` (narration disabled or AI not configured), no overlay is shown.

### To SPEC-010 (CLI & Config)
- Provides: `AiConfig` for CLI flag and TOML config deserialization
- Contract: `AiConfig` derives `serde::Deserialize`. The CLI exposes `--ai-url`, `--ai-model`, `--ai-key-env` (name of env var holding the API key, default `GOLF_AI_API_KEY`), `--ai-narrate`, `--ai-evolve`, `--ai-prompt`.

### To SPEC-002 (Rendering) -- Conversation Mode
- Provides: `ConversationState` management and `conversation_turn()` function
- Contract: The rendering layer captures user text input when conversation mode is active and calls `conversation_turn()`. The returned string is displayed in the overlay. Any embedded pattern/rule JSON is parsed and emitted as events.

## Integration Points

### API Key Management
API keys are never stored in the codebase or passed as CLI arguments directly. The flow:
1. User sets an environment variable (default `GOLF_AI_API_KEY`, configurable via `--ai-key-env`).
2. `AiConfig` reads the key from the environment at construction time.
3. For local LLMs (Ollama, llama.cpp), the API key is typically `None` -- the client sends requests without an `Authorization` header.

### Crate Dependency Graph
```
golf-ai  --> golf-entropy (for EntropySource trait, EntropyEvent)
         --> golf-core (for NdGrid read access, RuleSet, Cell)
         --> reqwest, serde_json, tokio (HTTP client infrastructure)

golf-cli --> golf-ai (optional, behind "ai" feature)
         --> golf-core
         --> golf-entropy
```

`golf-ai` depends on both `golf-core` and `golf-entropy`. This is intentional: it needs core types to read grid state and entropy types to produce events.

### Snapshot Publishing Integration
The simulation loop must be modified to publish snapshots. The integration point is in `SimulationEngine::step()` (SPEC-001):

```rust
// In the simulation loop (SPEC-001 engine or SPEC-010 CLI main loop):
if generation % snapshot_interval == 0 {
    let snapshot = build_snapshot(engine.grid(), engine.rules(), engine.generation());
    let _ = snapshot_tx.send(snapshot); // watch::send never blocks
}
```

This is a minimal addition to the simulation loop: one modulo check and one non-blocking send per N generations. The `build_snapshot()` call is the only cost.

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| LLM API unreachable | `reqwest` returns connection error | Log at Warn. Skip this cycle. Retry on next interval. Source stays in `Running` status. |
| LLM returns non-JSON or malformed JSON | `serde_json::from_str` fails | Log at Warn with raw response (truncated to 200 chars). Skip this cycle. |
| LLM returns coordinates outside grid bounds | Coordinate validation in `generate_pattern()` | Discard out-of-bounds coordinates. Keep valid ones. Log discarded count at Debug. |
| LLM suggests invalid rule values (e.g., birth={-1}) | Validation in `suggest_rule_evolution()` | Reject entire suggestion. Log at Warn. |
| API key missing or invalid | HTTP 401 response | Log at Warn. Set an internal flag to avoid spamming retries (exponential backoff: 30s, 60s, 120s, max 5min). |
| State channel not connected | `state_rx` is `None` | Evolution and narration tasks exit immediately with a Debug log. Pattern generation still works. |
| Conversation mode response contains no actionable content | Response parsed, no JSON blocks found | Display the text response as-is. No events emitted. This is normal -- the AI is just chatting. |
| Watch channel sender dropped (engine stopped) | `watch::Receiver::changed()` returns `Err` | AI background tasks exit cleanly. Source status set to `Stopped`. |
