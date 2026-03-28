# SPEC-008: AI Integration -- Tasks

## Task List

### T1: LlmClient trait and OpenAiCompatibleClient
- **Scope:** Create `crates/golf-ai/src/client.rs` and `crates/golf-ai/src/config.rs`. Define `LlmClient` trait, `ChatMessage`, `Role`, `CompletionConfig`, `ResponseFormat`. Implement `OpenAiCompatibleClient` using `reqwest` -- POST to `{base_url}/chat/completions`, parse streaming or non-streaming responses. Handle API key from environment. Add timeout handling (30s default).
- **AC coverage:** AC-4 (custom base URL support)
- **Estimate:** M

### T2: GridSnapshot and state observation
- **Scope:** Create `crates/golf-ai/src/snapshot.rs`. Define `GridSnapshot`, `RuleSetSummary`, `ClusterSummary`. Implement `build_snapshot()` -- cell counting, density calculation, cluster detection via downsampled flood-fill, age histogram bucketing. Ensure the function is fast (benchmark target: under 100 microseconds for 200x50 grid).
- **AC coverage:** Foundation for AC-2, AC-3, AC-6
- **Estimate:** M

### T3: Pattern generation
- **Scope:** Create `crates/golf-ai/src/pattern.rs`. Define `PatternResponse`. Implement `generate_pattern()` -- system prompt construction, JSON response parsing, coordinate validation and bounds clamping. Include prompt templates for common requests.
- **AC coverage:** AC-1 (glider prompt returns valid coordinates)
- **Estimate:** S

### T4: Rule evolution
- **Scope:** Create `crates/golf-ai/src/evolution.rs`. Define `RuleEvolutionResponse`, `EvolutionConfig`. Implement `suggest_rule_evolution()` -- stagnation detection (compare density across snapshots), prompt construction with current state context, response parsing, rule value validation.
- **AC coverage:** AC-2 (stagnant simulation triggers rule change suggestion)
- **Estimate:** M

### T5: Behavior narration
- **Scope:** Create `crates/golf-ai/src/narration.rs`. Define `NarrationConfig`. Implement `narrate_state()` -- prompt construction with snapshot context, response extraction. Set up the narration `mpsc` channel for the rendering layer.
- **AC coverage:** AC-3 (natural language descriptions appear in overlay)
- **Estimate:** S

### T6: AiEntropySource integration
- **Scope:** Create `crates/golf-ai/src/source.rs`. Implement `AiEntropySource` with `EntropySource` trait. Wire state channel (`watch::Receiver`). Spawn background tasks for evolution (interval loop) and narration (interval loop). Handle initial pattern generation on startup if configured. Implement `narration_rx()` accessor. Cancellation via `CancellationToken`.
- **AC coverage:** AC-5 (events flow through standard EntropySource channel)
- **Estimate:** M

### T7: Conversation mode
- **Scope:** Create `crates/golf-ai/src/conversation.rs`. Define `ConversationState`. Implement `conversation_turn()` -- history management, system prompt with current snapshot, response parsing for embedded JSON (pattern blocks, rule blocks). Implement history cap and oldest-message eviction.
- **AC coverage:** AC-6 (user types message, AI responds with simulation context)
- **Estimate:** M

### T8: Unit tests
- **Scope:** Test `build_snapshot()` with a known grid state (verify density, cluster count, age histogram). Test `generate_pattern()` response parsing with valid and malformed JSON. Test `suggest_rule_evolution()` validation (reject out-of-range values). Test `ConversationState` history eviction. Mock `LlmClient` for all tests (return canned responses).
- **AC coverage:** AC-1 through AC-6 (unit-level verification)
- **Estimate:** M

### T9: Integration test with local LLM
- **Scope:** Integration test that connects to a local Ollama instance (if available), sends a pattern generation prompt, and verifies valid coordinates are returned. Gate behind environment variable `GOLF_TEST_AI=1` and `GOLF_AI_TEST_URL`. Skip gracefully if the LLM is not reachable.
- **AC coverage:** AC-1, AC-4 (end-to-end with real LLM)
- **Estimate:** S

## Dependency Order

```
T1 (LlmClient) --> T2 (snapshot) --> T3 (pattern gen)
                                 --> T4 (rule evolution)
                                 --> T5 (narration)
                                     all of above --> T6 (AiEntropySource)
                                                 --> T7 (conversation)
                                                     both --> T8 (unit tests) --> T9 (integration test)
```

T1 is the foundation (all AI features need the HTTP client). T2 is needed by T4, T5, T6, and T7 (they all consume snapshots). T3, T4, and T5 are independent of each other. T6 wires the source together. T7 is independent of T6 but depends on T1-T5. T8 and T9 are sequential after everything else.
