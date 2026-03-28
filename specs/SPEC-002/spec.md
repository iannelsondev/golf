# SPEC-002: Rendering

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Rendering
**Dependencies:** SPEC-001
**Issue:** #2

## Problem Statement

The N-dimensional simulation state must be projected to 2D and rendered beautifully in the terminal at 60+ FPS. This means efficient differential rendering (only redraw changed cells), rich color schemes based on cell age, trail/fade effects for recently-dead cells, dimensional projection controls for higher-dimensional sims, and a responsive layout that adapts to terminal size. The renderer must never block the simulation loop.

## Functional Requirements

- **FR-1:** Render 2D grid slices to terminal using ratatui + crossterm with Unicode block characters for dense cell display
- **FR-2:** Cell age-based color gradients — newborn cells are bright, old cells shift through a configurable color spectrum
- **FR-3:** Death trail effect — recently-dead cells fade over N frames rather than disappearing instantly
- **FR-4:** Multiple built-in color palettes: `inferno` (fire), `ocean` (blue-cyan), `matrix` (green), `neon` (purple-pink), `grayscale`
- **FR-5:** FPS counter and simulation stats overlay (generation count, live cell count, dimensions, active entropy sources, current rule set)
- **FR-6:** N→2D projection modes for dimensions > 2: slice (2D cross-section at configurable depth), flatten (overlay layers with alpha), and rotation (animate through dimensional axes)
- **FR-7:** Configurable cell characters — full block, half block, braille dots, or custom Unicode characters

## Non-Functional Requirements

- **NFR-1:** Maintain 60 FPS at 200x50 2D grid on a standard terminal emulator
- **NFR-2:** Rendering must not block the simulation thread — use double-buffered frame state

## Acceptance Criteria

- **AC-1:** Given a running 2D simulation, when rendered, then cells display colors corresponding to their age using the selected palette
- **AC-2:** Given a cell that just died, when rendered over the next 5 frames, then it fades from dim to invisible
- **AC-3:** Given a 3D simulation in slice mode, when the user changes the Z-depth, then the displayed 2D slice updates to show that layer
- **AC-4:** Given a terminal resize event, when detected, then the grid display adapts within one frame without crashing
- **AC-5:** Given a running simulation, when the stats overlay is enabled, then generation count, live cells, dimensions, and FPS are displayed
- **AC-6:** Given braille rendering mode, when enabled, then each terminal cell represents a 2x4 sub-grid of simulation cells
- **AC-7:** Given a 200x50 grid, when benchmarked, then frame render time averages under 16ms (60 FPS)
