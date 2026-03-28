# SPEC-010: CLI & Config

**Status:** approved
**Spec Type:** api
**Feature Area:** CLI & Config
**Dependencies:** SPEC-001, SPEC-002, SPEC-003
**Issue:** #10

## Problem Statement

Golf needs a discoverable, composable CLI that exposes the full power of the framework through subcommands. Users should be able to stack entropy sources, select rules, configure rendering, and save presets — all from the command line or a TOML config file. The CLI is the primary user interface.

## Functional Requirements

- **FR-1:** `golf run` — start a simulation with configurable dimensions, rules, sources, and rendering options
- **FR-2:** `golf list-sources` — list available entropy sources and their status (available, unavailable + reason)
- **FR-3:** `golf demo` — curated demo modes showcasing different source/rule combinations (e.g., `golf demo gpu-fire`, `golf demo audio-pulse`, `golf demo 3d-slice`)
- **FR-4:** Source stacking via `--source` flags — `golf run --source random --source gpu --source audio` enables multiple simultaneous sources
- **FR-5:** TOML configuration file at `~/.config/golf/config.toml` with all runtime-configurable options, overridable by CLI flags
- **FR-6:** Preset system — save and load named presets (`golf preset save my-setup`, `golf run --preset my-setup`)
- **FR-7:** Runtime keybindings — pause/resume (space), step (n), cycle palette (c), toggle overlay (o), quit (q), zoom (±), pan (arrows)

## Non-Functional Requirements

- **NFR-1:** `golf --help` output must be clear, concise, and show common usage examples
- **NFR-2:** Invalid flag combinations produce helpful error messages, not panics

## Acceptance Criteria

- **AC-1:** Given `golf run --dimensions 2 --rules conway --source random`, when executed, then a 2D Conway simulation starts with random seeding
- **AC-2:** Given `golf run --dimensions 3 --rules 3d-life --projection slice --slice-depth 25`, when executed, then a 3D simulation is shown as a 2D slice at Z=25
- **AC-3:** Given `golf list-sources` on a system without CUDA, when executed, then GPU source shows "unavailable: CUDA runtime not found"
- **AC-4:** Given `golf demo gpu-fire`, when executed, then a pre-configured simulation starts with GPU entropy and inferno palette
- **AC-5:** Given a TOML config setting `palette = "ocean"` and CLI flag `--palette neon`, when run, then the CLI flag wins
- **AC-6:** Given `golf preset save night-mode` followed by `golf run --preset night-mode`, when executed, then the saved settings are restored
- **AC-7:** Given a running simulation, when the user presses 'c', then the color palette cycles to the next option
