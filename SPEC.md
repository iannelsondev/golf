# golf — Game of Life Framework

## Vision

A terminal-native Game of Life framework that turns Conway's cellular automaton into a living, breathing entropy visualizer. Feed it GPU VRAM noise, microphone input, network traffic, AI-generated patterns, or any stream of chaos — and watch order emerge from disorder in real time. Inspired by `conway-screensaver` but built for people who want their terminal to feel alive.

## Goals

- Render Conway's Game of Life at 60+ FPS with smooth, beautiful terminal output
- Support pluggable entropy sources that seed and perturb simulations in real time
- Read raw GPU VRAM via CUDA as a first-class entropy source
- Capture audio input (microphone, system audio) and map waveforms/FFT to cell patterns
- Integrate with LLMs to generate patterns from text, mutate rulesets, and narrate emergent behavior
- Support classic Game of Life file formats (RLE, Life 1.06, plaintext) for pattern loading
- Provide a subcommand-based CLI with discoverable, composable options
- Make it mesmerizing — color gradients, cell age heatmaps, trail effects, smooth transitions

## Non-Goals

- GUI or web interface — this is terminal-only
- Multiplayer or networked simulation sync
- Hand-editing patterns in a visual editor (load from files instead)
- Scientific accuracy or research-grade simulation (we optimize for aesthetics)
- Cross-platform GPU support in v1 (CUDA first, Vulkan later)
- Windows support (Linux-first, macOS where it works)

## Architecture

Single Rust binary (`golf`) with subcommands. Workspace layout:

```
golf/
  crates/
    golf-core/        # Grid engine, rules, cell types, simulation loop
    golf-render/      # Terminal rendering (ratatui), color schemes, effects
    golf-entropy/     # Entropy source trait + implementations (CUDA, audio, network, etc.)
    golf-ai/          # LLM integration (pattern gen, rule mutation, narration)
    golf-formats/     # File format parsers (RLE, Life 1.06, plaintext)
    golf-cli/         # Clap CLI, subcommands, config
```

The core loop: entropy sources produce `EntropyEvent`s (cell injections, rule mutations, energy pulses). The simulation engine consumes events, steps the grid, and the renderer paints each frame. Sources run on background threads/tasks and push events through channels.

**Key crate choices:**
- `ratatui` + `crossterm` — terminal rendering
- `clap` — CLI parsing
- `tokio` — async runtime (audio streams, network, AI API calls)
- `cudarc` — CUDA device access for VRAM reads
- `cpal` — cross-platform audio capture
- `rodio` or raw `cpal` — audio analysis (FFT via `rustfft`)

## Feature Areas

- **Core Engine** — Grid data structure, B3/S23 rules, step function, toroidal/bounded edges, configurable grid size (SPEC-001)
- **Rendering** — ratatui-based renderer, cell age coloring, heatmaps, trail/fade effects, FPS counter, multiple color schemes (SPEC-002)
- **Entropy Trait** — Pluggable `EntropySource` trait, `EntropyEvent` types, real-time event channel, source registry (SPEC-003)
- **Classic Sources** — Random init, file pattern loading (RLE, Life 1.06, plaintext), procedural generators (noise, fractals, glider guns) (SPEC-004)
- **GPU Entropy** — CUDA VRAM reads, memory region selection, bit-to-cell mapping, entropy density control (SPEC-005)
- **Audio Entropy** — Microphone/system audio capture, FFT spectral analysis, waveform-to-grid mapping, beat detection → cell pulses (SPEC-006)
- **Network Entropy** — Packet capture or socket data as entropy, DNS query visualization, ping latency mapping (SPEC-007)
- **AI Integration** — LLM pattern generation from text prompts, rule mutation suggestions, emergent behavior narration, interactive chat overlay (SPEC-008)
- **Visual Effects** — Smooth transitions between states, particle trails, glow effects, zoom/pan, split-screen multi-sim (SPEC-009)
- **CLI & Config** — Subcommands (`golf run`, `golf list-sources`, `golf demo`), TOML config, preset management, `--source` stacking (SPEC-010)

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
| SPEC-001 | Core Engine | draft | Grid engine, rules, simulation step |
| SPEC-002 | Rendering | draft | Terminal rendering, color, effects |
| SPEC-003 | Entropy Trait | draft | Pluggable entropy source abstraction |
| SPEC-004 | Classic Sources | draft | Random, file patterns, procedural generators |
| SPEC-005 | GPU Entropy | draft | CUDA VRAM entropy source |
| SPEC-006 | Audio Entropy | draft | Microphone/audio FFT entropy source |
| SPEC-007 | Network Entropy | draft | Network traffic entropy source |
| SPEC-008 | AI Integration | draft | LLM pattern generation and narration |
| SPEC-009 | Visual Effects | draft | Transitions, trails, glow, split-screen |
| SPEC-010 | CLI & Config | draft | Subcommands, TOML config, presets |
