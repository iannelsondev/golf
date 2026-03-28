# golf — Game of Life Framework

## Vision

A terminal-native cellular automata framework built on N-dimensional set theory. The engine models automata in arbitrary-dimensional spaces — 2D Conway's Game of Life is just the default projection. Feed it GPU VRAM noise, microphone input, network traffic, AI-generated patterns, or any stream of chaos — and watch order emerge from disorder in real time. Inspired by `conway-screensaver` and the ideas in "Cellular Automata: Cyber Creepy Crawlies" (Nelson & Looper, ISEF 2000, AAAI Award), which mapped traditional AI concepts back to automata using multi-dimensional set theory. Golf carries that work forward with modern Rust, real-time entropy sources, and LLM integration as the bridge between classical AI↔automata correspondence and contemporary generative AI.

## Goals

- Model cellular automata in N-dimensional spaces with set-theoretic rule definitions — 2D is the default, not the limit
- Render projections of N-dimensional state to the terminal at 60+ FPS with beautiful output
- Support pluggable entropy sources that seed and perturb simulations in real time
- Read raw GPU VRAM via CUDA as a first-class entropy source
- Capture audio input (microphone, system audio) and map waveforms/FFT to cell patterns
- Bridge classical AI↔automata correspondence with modern LLMs — pattern generation, rule evolution, behavior narration
- Support classic Game of Life file formats (RLE, Life 1.06, plaintext) for pattern loading
- Provide a subcommand-based CLI with discoverable, composable options
- Make it mesmerizing — color gradients, cell age heatmaps, trail effects, smooth transitions

## Non-Goals

- GUI or web interface — this is terminal-only
- Multiplayer or networked simulation sync
- Hand-editing patterns in a visual editor (load from files instead)
- Scientific accuracy or research-grade simulation (we optimize for aesthetics over formal proofs)
- Cross-platform GPU support in v1 (CUDA first, Vulkan later)
- Windows support (Linux-first, macOS where it works)

## Architecture

Single Rust binary (`golf`) with subcommands. Workspace layout:

```
golf/
  crates/
    golf-core/        # N-dimensional grid engine, set-theoretic rules, simulation loop
    golf-render/      # Terminal rendering (ratatui), N→2D projection, color schemes, effects
    golf-entropy/     # Entropy source trait + implementations (CUDA, audio, network, etc.)
    golf-ai/          # LLM↔automata bridge (pattern gen, rule evolution, narration)
    golf-formats/     # File format parsers (RLE, Life 1.06, plaintext)
    golf-cli/         # Clap CLI, subcommands, config
```

**Core abstraction:** The grid is an N-dimensional space where cells are elements of a set. Neighborhoods are defined as set operations (Moore, von Neumann, or custom N-dimensional neighborhoods). Rules are predicates over neighborhood cardinality — B3/S23 is just `birth: {3}, survival: {2, 3}` in the 2D Moore case. This generalizes cleanly to higher dimensions.

**Core loop:** Entropy sources produce `EntropyEvent`s (cell injections, rule mutations, energy pulses, dimensional shifts). The simulation engine consumes events, steps the grid, and the renderer projects the state to 2D and paints each frame. Sources run on background tasks and push events through channels.

**Dimensional projection:** Higher-dimensional simulations are projected to 2D for terminal display — slicing (show a 2D cross-section), flattening (overlay multiple layers with alpha blending), or rotation (animate through dimensional axes).

**Key crate choices:**
- `ratatui` + `crossterm` — terminal rendering
- `clap` — CLI parsing
- `tokio` — async runtime (audio streams, network, AI API calls)
- `cudarc` — CUDA device access for VRAM reads
- `cpal` — cross-platform audio capture
- `rodio` or raw `cpal` — audio analysis (FFT via `rustfft`)

## Feature Areas

- **Core Engine** — N-dimensional grid, set-theoretic rule system, step function, toroidal/bounded edges, cell age tracking (SPEC-001)
- **Rendering** — ratatui-based renderer, N→2D projection, cell age coloring, heatmaps, trail/fade effects, FPS counter, color schemes (SPEC-002)
- **Entropy Trait** — Pluggable `EntropySource` trait, `EntropyEvent` types, real-time event channel, source registry (SPEC-003)
- **Classic Sources** — Random init, file pattern loading (RLE, Life 1.06, plaintext), procedural generators (noise, fractals, glider guns) (SPEC-004)
- **GPU Entropy** — CUDA VRAM reads, memory region selection, bit-to-cell mapping, entropy density control (SPEC-005)
- **Audio Entropy** — Microphone/system audio capture, FFT spectral analysis, waveform-to-grid mapping, beat detection → cell pulses (SPEC-006)
- **Network Entropy** — Packet capture or socket data as entropy, DNS query visualization, ping latency mapping (SPEC-007)
- **AI Integration** — LLM↔automata bridge: pattern generation, rule evolution, behavior narration, classical AI correspondence (SPEC-008)
- **Visual Effects** — Smooth transitions, particle trails, glow effects, zoom/pan, split-screen, dimensional projection controls (SPEC-009)
- **CLI & Config** — Subcommands (`golf run`, `golf list-sources`, `golf demo`), TOML config, preset management, `--source` stacking (SPEC-010)
- **MCP Interface** — MCP server exposing simulation control, entropy injection, rule mutation, and state queries as tools for LLM agents and editors (SPEC-011)

## Constraints

- Rust 2024 edition, tokio async runtime
- 60 FPS minimum rendering target at 1080p terminal size (approx 200x50 cells)
- Must not require root — GPU access via user-space CUDA runtime
- Graceful degradation: if CUDA unavailable, skip GPU source with a warning (not a crash)
- If no microphone, skip audio source with a warning
- All entropy sources are optional — the binary works with zero external dependencies (falls back to random)
- Terminal must remain responsive — simulation never blocks input handling

## Spec Index

| Spec | Feature Area | Status | Summary |
|------|-------------|--------|---------|
| SPEC-001 | Core Engine | approved | N-dimensional grid engine, set-theoretic rules |
| SPEC-002 | Rendering | approved | Terminal rendering, N→2D projection, color, effects |
| SPEC-003 | Entropy Trait | approved | Pluggable entropy source abstraction |
| SPEC-004 | Classic Sources | approved | Random, file patterns, procedural generators |
| SPEC-005 | GPU Entropy | approved | CUDA VRAM entropy source |
| SPEC-006 | Audio Entropy | approved | Microphone/audio FFT entropy source |
| SPEC-007 | Network Entropy | approved | Network traffic entropy source |
| SPEC-008 | AI Integration | approved | LLM↔automata bridge, pattern gen, rule evolution |
| SPEC-009 | Visual Effects | approved | Transitions, trails, glow, split-screen |
| SPEC-010 | CLI & Config | approved | Subcommands, TOML config, presets |
| SPEC-011 | MCP Interface | approved | MCP server for programmatic simulation control |

## Repository

| Field | Value |
|-------|-------|
| URL | https://github.com/iannelsondev/golf |
| Provider | github |
| Default Branch | main |
| Issue Tracker | github |
| PR/MR Strategy | squash-merge |
| Branch Naming | spec/{spec-id}/{short-title} |
| CI Required | false |
| Auto-merge | false |
| Labels | spec, auto-generated |
