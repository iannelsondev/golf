# SPEC-008: AI Integration

**Status:** approved
**Spec Type:** behavior
**Feature Area:** AI Integration
**Dependencies:** SPEC-001, SPEC-003
**Issue:** #8

## Problem Statement

"Cellular Automata: Cyber Creepy Crawlies" (Nelson & Looper, ISEF 2000) mapped traditional AI concepts back to automata. Golf carries this forward: LLMs are the modern bridge between human intent and automata behavior. An AI can generate patterns from natural language, evolve rulesets by observing emergent behavior, narrate what the simulation is doing in real time, and explore the classical AI↔automata correspondence — treating the cellular automaton as a computational substrate that AI can reason about and manipulate.

## Functional Requirements

- **FR-1:** Pattern generation from text — send a natural language prompt ("create a spaceship factory") to an LLM, receive N-dimensional cell coordinates as a pattern to inject
- **FR-2:** Rule evolution — AI observes simulation state over time, suggests rule mutations (birth/survival set changes) that produce more interesting emergent behavior
- **FR-3:** Behavior narration — AI periodically analyzes the grid state and produces natural language descriptions of what's happening ("a cluster of oscillators is forming near the entropy injection point")
- **FR-4:** Pluggable LLM backend — support any OpenAI-compatible API endpoint (local llama.cpp, Ollama, cloud providers) via configurable base URL and API key
- **FR-5:** AI as entropy source — the AI integration implements `EntropySource`, pushing pattern injections and rule mutations as `EntropyEvent`s through the standard channel
- **FR-6:** Conversation mode — interactive prompt overlay where the user can chat with the AI about the simulation state while it runs

## Non-Functional Requirements

- **NFR-1:** AI API calls must be fully async and never block the simulation or rendering loops
- **NFR-2:** AI responses are cached/debounced — no more than 1 narration request per 5 seconds, 1 rule suggestion per 30 seconds

## Acceptance Criteria

- **AC-1:** Given the prompt "create a glider", when sent to the LLM, then coordinates for a valid glider pattern are returned and injected onto the grid
- **AC-2:** Given a stagnant simulation (mostly still lifes), when rule evolution is active, then the AI suggests a rule change that increases activity
- **AC-3:** Given a running simulation, when narration is enabled, then natural language descriptions appear in the overlay every 5+ seconds
- **AC-4:** Given an OpenAI-compatible API at a custom base URL, when configured, then all AI features use that endpoint
- **AC-5:** Given the AI entropy source active, when it generates events, then they flow through the standard `EntropySource` channel like any other source
- **AC-6:** Given conversation mode active, when the user types a message, then the AI responds with context about the current simulation state

## Constraints

- No hardcoded API keys — all credentials via environment variables or config file
- AI features are entirely optional — golf runs without any AI configuration
