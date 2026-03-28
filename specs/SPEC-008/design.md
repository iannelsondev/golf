# SPEC-008: AI Integration -- Design

## Signatures

### Files Created or Modified

- `crates/golf-ai/src/lib.rs` -- crate root, re-exports public API
- `crates/golf-ai/src/client.rs` -- `LlmClient` trait and `OpenAiCompatibleClient` implementation
- `crates/golf-ai/src/source.rs` -- `AiEntropySource` implementing `EntropySource`
- `crates/golf-ai/src/pattern.rs` -- pattern generation prompt construction and response parsing
- `crates/golf-ai/src/evolution.rs` -- rule evolution logic: state snapshot to rule mutation suggestions
- `crates/golf-ai/src/narration.rs` -- behavior narration: state snapshot to natural language
- `crates/golf-ai/src/conversation.rs` -- interactive conversation mode
- `crates/golf-ai/src/snapshot.rs` -- grid state snapshot serialization for LLM context
- `crates/golf-ai/src/config.rs` -- `AiConfig` and related configuration types
- `crates/golf-ai/Cargo.toml` -- crate manifest with `reqwest`, `serde_json`, `tokio`

### Types Introduced

- `LlmClient` -- trait abstracting LLM API communication
  ```rust
  #[async_trait]
  pub trait LlmClient: Send + Sync {
      async fn complete(&self, messages: Vec<ChatMessage>, config: &CompletionConfig) -> Result<String>;
      async fn complete_json<T: DeserializeOwned>(&self, messages: Vec<ChatMessage>, config: &CompletionConfig) -> Result<T>;
  }
  ```
- `OpenAiCompatibleClient` -- implementation for any OpenAI-compatible API
  ```rust
  pub struct OpenAiCompatibleClient {
      http: reqwest::Client,
      base_url: String,
      api_key: Option<String>,
      model: String,
  }
  ```
- `ChatMessage` -- conversation message
  ```rust
  pub struct ChatMessage {
      pub role: Role,
      pub content: String,
  }
  pub enum Role { System, User, Assistant }
  ```
- `CompletionConfig` -- per-request LLM parameters
  ```rust
  pub struct CompletionConfig {
      pub temperature: f32,       // default 0.7
      pub max_tokens: u32,        // default 1024
      pub response_format: Option<ResponseFormat>,
  }
  pub enum ResponseFormat { Text, Json }
  ```
- `AiEntropySource` -- entropy source that wraps all AI capabilities
  ```rust
  pub struct AiEntropySource {
      client: Box<dyn LlmClient>,
      config: AiConfig,
      status: SourceStatus,
      cancel: Option<CancellationToken>,
      state_rx: Option<watch::Receiver<GridSnapshot>>,
  }
  ```
- `AiConfig` -- top-level AI configuration
  ```rust
  pub struct AiConfig {
      pub base_url: String,               // e.g., "http://localhost:11434/v1"
      pub api_key: Option<String>,         // from env var or config, never hardcoded
      pub model: String,                   // e.g., "llama3", "gpt-4o"
      pub pattern_gen: PatternGenConfig,
      pub evolution: EvolutionConfig,
      pub narration: NarrationConfig,
  }
  ```
- `PatternGenConfig`
  ```rust
  pub struct PatternGenConfig {
      pub enabled: bool,
      pub initial_prompt: Option<String>,  // run this prompt at startup
  }
  ```
- `EvolutionConfig`
  ```rust
  pub struct EvolutionConfig {
      pub enabled: bool,
      pub interval: Duration,             // default 30s, minimum 30s
      pub stagnation_threshold: f64,      // fraction of cells unchanged between snapshots
  }
  ```
- `NarrationConfig`
  ```rust
  pub struct NarrationConfig {
      pub enabled: bool,
      pub interval: Duration,             // default 10s, minimum 5s
  }
  ```
- `GridSnapshot` -- compact representation of grid state for LLM context
  ```rust
  pub struct GridSnapshot {
      pub extents: Vec<usize>,
      pub generation: u64,
      pub live_cell_count: usize,
      pub total_cells: usize,
      pub density: f64,
      pub rules: RuleSetSummary,
      pub clusters: Vec<ClusterSummary>,   // regions of high activity
      pub age_histogram: Vec<(u16, usize)>, // (age_bucket, count)
  }
  ```
- `RuleSetSummary` -- serializable rule description for LLM prompts
  ```rust
  pub struct RuleSetSummary {
      pub birth: Vec<u32>,
      pub survival: Vec<u32>,
      pub neighborhood: String,  // "moore" or "vonneumann"
  }
  ```
- `ClusterSummary` -- a region of high cell density
  ```rust
  pub struct ClusterSummary {
      pub center: Vec<usize>,
      pub radius: usize,
      pub live_cells: usize,
      pub avg_age: f64,
  }
  ```
- `PatternResponse` -- parsed LLM response for pattern generation
  ```rust
  pub struct PatternResponse {
      pub cells: Vec<Vec<usize>>,         // coordinates to set alive
      pub description: Option<String>,     // LLM's description of the pattern
  }
  ```
- `RuleEvolutionResponse` -- parsed LLM response for rule suggestions
  ```rust
  pub struct RuleEvolutionResponse {
      pub birth: Vec<u32>,
      pub survival: Vec<u32>,
      pub reasoning: String,
  }
  ```
- `ConversationState` -- tracks the interactive conversation
  ```rust
  pub struct ConversationState {
      pub history: Vec<ChatMessage>,
      pub max_history: usize,             // default 20 messages, oldest dropped
  }
  ```

### Functions / Hooks / Contracts

- `OpenAiCompatibleClient::new(base_url: &str, api_key: Option<String>, model: &str) -> Self`
- `AiEntropySource::new(client: Box<dyn LlmClient>, config: AiConfig) -> Self`
- `AiEntropySource` implements `EntropySource`:
  - `fn name(&self) -> &str` -- returns `"ai"`
  - `fn kind(&self) -> &str` -- returns `"ai"`
  - `async fn start(&mut self) -> Result<Receiver<EntropyEvent>>` -- spawns background tasks for evolution and narration
  - `async fn stop(&mut self) -> Result<()>` -- cancels tasks
  - `fn metadata(&self) -> SourceMetadata`
- `AiEntropySource::set_state_channel(&mut self, rx: watch::Receiver<GridSnapshot>)` -- connects the AI to live grid state
- `fn generate_pattern(client: &dyn LlmClient, prompt: &str, grid_extents: &[usize]) -> Result<PatternResponse>` -- one-shot pattern generation
- `fn suggest_rule_evolution(client: &dyn LlmClient, snapshot: &GridSnapshot) -> Result<RuleEvolutionResponse>` -- one-shot rule suggestion
- `fn narrate_state(client: &dyn LlmClient, snapshot: &GridSnapshot) -> Result<String>` -- one-shot narration
- `fn build_snapshot(grid: &NdGrid<N>, rules: &RuleSet, generation: u64) -> GridSnapshot` -- extract snapshot from live grid (called from engine thread, must be fast)
- `ConversationState::new(system_prompt: String) -> Self`
- `ConversationState::add_user_message(&mut self, msg: String)`
- `ConversationState::add_assistant_message(&mut self, msg: String)`
- `async fn conversation_turn(client: &dyn LlmClient, state: &mut ConversationState, snapshot: &GridSnapshot) -> Result<String>` -- sends conversation history + current snapshot, returns AI response

## Architecture

### The AI-Automata Bridge

This design builds on the central insight from "Cellular Automata: Cyber Creepy Crawlies" (Nelson & Looper, ISEF 2000, AAAI Award): traditional AI concepts -- search, optimization, pattern recognition, learning -- have direct correspondences in cellular automata behavior. Gliders perform search. Oscillators are fixed points. Emergent structures are solutions found by the automaton's implicit computation.

Golf extends this correspondence to modern generative AI. Where the original paper mapped classical AI concepts to automata post-hoc, the LLM integration creates a bidirectional bridge:

- **Automata to AI:** The simulation state is serialized into a `GridSnapshot` and sent to the LLM as structured context. The LLM observes patterns, identifies structures, and reasons about the automaton's behavior.
- **AI to Automata:** The LLM's responses are parsed into `EntropyEvent`s -- cell coordinates, rule mutations -- that are injected back into the simulation through the standard entropy channel.

The LLM becomes a participant in the automaton's evolution, not just an observer. It can nudge the simulation toward interesting behavior, name what it sees, and respond to the user's intent expressed in natural language.

### State Observation via watch Channel

The simulation engine publishes grid state via a `tokio::sync::watch` channel:

```
SimulationEngine --build_snapshot()--> watch::Sender<GridSnapshot>
                                            |
                                       watch::Receiver (cloned to AiEntropySource)
```

The engine calls `build_snapshot()` once per N generations (configurable, default every 10 generations) and sends it through the watch channel. The watch channel always holds the latest snapshot -- the AI reads it when needed, never blocking the engine.

`build_snapshot()` must be fast because it runs on the simulation thread. It computes:
- Live cell count (single pass over the cell buffer)
- Density (live / total)
- Rule set summary (copy birth/survival sets)
- Cluster detection (simple grid scan with flood-fill on a downsampled grid)
- Age histogram (bucket ages into 10 ranges)

For a 200x50 grid (10,000 cells), this takes microseconds.

### Pattern Generation

When pattern generation is requested (via initial prompt, conversation, or periodic trigger):

1. Construct a system prompt explaining the coordinate system, grid extents, and that the response must be a JSON array of coordinate arrays.
2. Append the user's natural language request.
3. Call `client.complete_json::<PatternResponse>()`.
4. Validate returned coordinates are within grid bounds. Discard out-of-bounds coordinates with a log warning.
5. Emit `ApplyPattern { cells, offset: vec![0; N] }` as an `EntropyEvent`.

System prompt template:
```
You are controlling a cellular automaton on a grid with extents {extents}.
Coordinates are zero-indexed arrays of {N} integers.
Respond with JSON: {"cells": [[x, y], ...], "description": "what this pattern is"}
Create a pattern based on the user's request. Use your knowledge of
Game of Life patterns (gliders, spaceships, oscillators, still lifes)
to produce valid, interesting patterns.
```

### Rule Evolution

When evolution is enabled, a background task runs on an interval:

1. Read the latest `GridSnapshot` from the watch channel.
2. Check for stagnation: if `density` has not changed by more than `stagnation_threshold` (default 0.05) since the last evolution check, the simulation is considered stagnant.
3. If stagnant (or on first run), call `suggest_rule_evolution()`.
4. The prompt includes the current rules, density, generation count, and cluster information. It asks the LLM to suggest a birth/survival set change that would increase emergent complexity.
5. Parse the response as `RuleEvolutionResponse`.
6. Validate: birth and survival values must be in range `0..=(3^N - 1)` (max possible neighbors). Reject values outside this range.
7. Emit `MutateRules(RuleSet { birth, survival, neighborhood })` as an `EntropyEvent`.

The debounce (minimum 30 seconds between suggestions, per NFR-2) is enforced by the task's sleep interval.

### Behavior Narration

When narration is enabled, a background task runs on an interval:

1. Read the latest `GridSnapshot`.
2. Call `narrate_state()` with a system prompt that describes the simulation context and asks for a 1-2 sentence description of what is happening.
3. The narration text is not an `EntropyEvent` -- it does not affect the simulation. Instead, it is sent through a separate `tokio::sync::mpsc` channel to the rendering layer, which overlays it on the terminal display.
4. The narration channel is exposed via `AiEntropySource::narration_rx() -> Option<Receiver<String>>`.

Narration prompt template:
```
You are observing a cellular automaton. Current state:
- Grid: {extents}, generation {generation}
- {live_cell_count} live cells ({density:.1%} density)
- Rules: B{birth}/S{survival} ({neighborhood})
- Clusters: {cluster_descriptions}

Describe what you see in 1-2 sentences. Be poetic but accurate.
Reference specific structures if you can identify them (gliders,
oscillators, still lifes, chaotic regions).
```

### Conversation Mode

Conversation mode is activated by the user via a keyboard shortcut in the terminal UI. When active:

1. A text input overlay appears at the bottom of the terminal.
2. User types a message and presses Enter.
3. `conversation_turn()` is called with the conversation history and current snapshot.
4. The system prompt includes the current grid state and explains the AI's capabilities (it can suggest patterns, explain behavior, modify rules).
5. The AI's response is displayed in the overlay.
6. If the AI's response contains a JSON pattern block (detected by regex), it is parsed and injected as an `ApplyPattern` event.
7. If the AI's response contains a JSON rule block, it is parsed and injected as a `MutateRules` event.

Conversation history is capped at `max_history` messages (default 20). When exceeded, the oldest non-system messages are dropped. The system prompt (with current snapshot) is always regenerated fresh for each turn.

### Debouncing and Rate Control

Per NFR-2:
- Narration: minimum 5 seconds between requests (enforced by task sleep interval, configurable via `NarrationConfig::interval`)
- Rule evolution: minimum 30 seconds between requests (enforced by task sleep interval)
- Pattern generation: no automatic debounce (it is user-triggered or one-shot at startup)
- Conversation: no debounce (user controls the pace)

All API calls use `tokio::time::timeout(Duration::from_secs(30), ...)` to prevent hung requests from blocking the background tasks.

### Error Handling

LLM API calls are inherently unreliable. The design handles errors without affecting the simulation:

- **Network error / timeout:** Log at Warn level. Skip this cycle. Retry on the next interval.
- **Malformed JSON response:** Log at Warn level with the raw response. Skip this cycle.
- **Out-of-bounds coordinates:** Discard individual bad coordinates, keep valid ones. Log at Debug.
- **Invalid rule values:** Reject the entire suggestion. Log at Warn with the invalid values.
- **API key missing:** `start()` succeeds (the source is technically running), but the first API call fails and sets status to `Error`. The source continues attempting on each interval in case the key is provided later via environment.

The simulation never blocks or crashes due to AI failures. The AI is a "nice to have" entropy source that gracefully degrades to producing no events.

### Feature Gate

The entire `golf-ai` crate is a separate workspace member, not feature-gated within `golf-entropy`. The CLI conditionally depends on `golf-ai`:

```toml
# crates/golf-cli/Cargo.toml
[features]
ai = ["dep:golf-ai"]

[dependencies]
golf-ai = { path = "../golf-ai", optional = true }
```

When the `ai` feature is not enabled, the AI source is not available and conversation mode is not compiled.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| LLM returns invalid JSON despite JSON mode request | Pattern generation and rule evolution fail | Medium -- depends on model quality | Retry once with a stricter prompt. If still invalid, skip. Log the raw response for debugging. |
| LLM hallucinates Game of Life patterns that are not valid | Injected patterns are meaningless | Medium | Validate coordinates are in bounds. The simulation will naturally eliminate non-viable patterns -- this is a feature, not a bug. |
| LLM API latency is high (seconds) | Narration and evolution feel sluggish | High for cloud APIs, Low for local | All calls are async with timeouts. The simulation runs independently. Latency only affects how often AI features produce output. |
| API key exposure | Security risk | Low -- never hardcoded | Keys come from environment variables or config files only. Config files should be in `~/.config/golf/` with restricted permissions. |
| Token costs for cloud APIs | Unexpected bills | Medium | Debounce intervals limit request rate. Grid snapshots are compact (not raw cell data). Document expected token usage per hour in README. |
| `build_snapshot()` cluster detection is slow for large grids | Simulation stutter every N generations | Low -- downsampled flood-fill | Cluster detection operates on a 4x-downsampled grid. For 200x50, that is 50x12 = 600 cells. Flood-fill on 600 cells is sub-microsecond. |

## ADR-008-1: watch Channel over Polling for State Observation

**Status:** Accepted

**Context:** The AI needs periodic snapshots of the grid state. Options: (a) the AI task polls the engine via a shared reference, (b) the engine pushes snapshots through a channel.

**Decision:** Use `tokio::sync::watch` for state publication. The engine sends snapshots; the AI reads the latest when needed.

**Consequences:**
- Positive: No shared mutable state between the engine and AI. The watch channel always holds exactly one value (the latest snapshot). Multiple AI tasks can clone the receiver. The engine controls snapshot frequency.
- Negative: Snapshot construction has a cost (cell counting, cluster detection). This cost is paid on the simulation thread.
- Mitigation: Snapshots are built every N generations (default 10), not every step. The snapshot construction is O(cells) which is fast for target grid sizes.

## ADR-008-2: Separate Crate over Feature Gate in golf-entropy

**Status:** Accepted

**Context:** AI integration could be a feature-gated module within `golf-entropy` (like audio and network), or a separate crate (`golf-ai`).

**Decision:** Separate `golf-ai` crate.

**Consequences:**
- Positive: Clean dependency boundary. `golf-ai` depends on `reqwest`, `serde_json`, and HTTP client infrastructure that is unrelated to other entropy sources. Keeps `golf-entropy` focused on the trait and lightweight sources.
- Negative: One more crate in the workspace. The AI source still implements `EntropySource` from `golf-entropy`, creating a dependency from `golf-ai` to `golf-entropy`.
- Mitigation: The workspace already has 6 crates. One more is not significant. The cross-crate dependency is one-directional and clean.

## ADR-008-3: Compact GridSnapshot over Raw Cell Data for LLM Context

**Status:** Accepted

**Context:** The LLM needs to "see" the grid state. Options: (a) send the raw cell buffer (potentially thousands of values), (b) send a compact statistical summary.

**Decision:** Send a `GridSnapshot` containing aggregate statistics (density, cluster summaries, age histogram, rule description) rather than raw cell data.

**Consequences:**
- Positive: Dramatically fewer tokens per request. A 200x50 grid has 10,000 cells; the snapshot is approximately 200-300 tokens. LLMs reason better about summaries than raw arrays.
- Negative: The LLM cannot see individual cells or precise pattern shapes. It cannot identify specific known patterns (gliders, etc.) from statistics alone.
- Mitigation: Cluster summaries provide spatial information. For pattern identification, a future enhancement could render a small ASCII representation of active regions and include it in the prompt. For v1, aggregate statistics are sufficient for narration and rule evolution.
