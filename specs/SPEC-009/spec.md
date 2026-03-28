# SPEC-009: Visual Effects

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Visual Effects
**Dependencies:** SPEC-001, SPEC-002
**Issue:** #9

## Problem Statement

Beyond basic cell rendering, the simulation needs visual effects that make it mesmerizing to watch. Smooth transitions, particle trails, glow effects around high-activity regions, zoom/pan controls, and split-screen mode for comparing simulations or viewing multiple dimensional slices side by side. These effects transform a cellular automaton from a math exercise into something you leave running in a terminal all day.

## Functional Requirements

- **FR-1:** Smooth cell transitions — interpolate between states over multiple frames rather than hard on/off switching
- **FR-2:** Glow effect — high-density cell regions emit a visual glow that bleeds into neighboring terminal cells using background color blending
- **FR-3:** Zoom and pan — keyboard controls to zoom into a region of the grid and pan around, with smooth scrolling
- **FR-4:** Split-screen mode — display 2+ simulations side by side (different rules, different entropy sources, or different dimensional slices of the same sim)
- **FR-5:** Entropy source visualization — show where entropy events are being injected as brief flashes or colored markers distinct from normal cells
- **FR-6:** Configurable animation speed — separate simulation speed from visual animation speed (slow-motion replay of fast simulation)

## Non-Functional Requirements

- **NFR-1:** Effects must not drop frame rate below 30 FPS even when stacked
- **NFR-2:** All effects are individually toggleable at runtime via keybinds

## Acceptance Criteria

- **AC-1:** Given smooth transitions enabled, when a cell is born, then it brightens over 3 frames rather than appearing instantly
- **AC-2:** Given a high-density cluster of cells, when glow is enabled, then adjacent empty cells show a dim background color
- **AC-3:** Given zoom mode at 2x, when active, then a quarter of the grid fills the terminal at double resolution
- **AC-4:** Given split-screen with two sims, when both are running, then each occupies half the terminal and runs independently
- **AC-5:** Given an entropy injection event, when visualized, then a brief colored flash appears at the injection coordinates
- **AC-6:** Given animation speed set to 0.5x with sim speed at 60 steps/sec, when running, then the visual updates at 30 FPS showing every other frame

## Constraints

- Effects stack — all combinations must work together without visual artifacts
- Split-screen limited to 4 panes maximum (terminal real estate constraint)
