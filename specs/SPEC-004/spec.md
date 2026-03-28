# SPEC-004: Classic Sources

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Classic Sources
**Dependencies:** SPEC-001, SPEC-003
**Issue:** #4

## Problem Statement

Before exotic entropy sources like GPU VRAM or audio, the framework needs solid fundamentals: random initialization, loading patterns from standard Game of Life file formats, and procedural pattern generators. These are the sources that work everywhere with zero external dependencies.

## Functional Requirements

- **FR-1:** Random entropy source — configurable density (0.0-1.0), optional periodic re-seeding at configurable intervals
- **FR-2:** File pattern loader supporting RLE (Run Length Encoded) format — the most common Game of Life pattern exchange format
- **FR-3:** File pattern loader supporting Life 1.06 plaintext format
- **FR-4:** Procedural glider gun source — periodically injects Gosper glider guns at random positions to prevent stagnation (inspired by conway-screensaver's infinite mode)
- **FR-5:** Procedural noise source — Perlin/simplex noise mapped to cell birth probability, scrolling through noise space over time
- **FR-6:** Pattern library — bundled collection of classic patterns (glider, LWSS, pulsar, pentadecathlon, R-pentomino) loadable by name

## Non-Functional Requirements

- **NFR-1:** RLE parser must handle files up to 1MB without excessive memory allocation
- **NFR-2:** Pattern loading must complete in under 10ms for standard library patterns

## Acceptance Criteria

- **AC-1:** Given a random source with density 0.3, when it generates initial state, then approximately 30% of cells are alive (within 5% tolerance)
- **AC-2:** Given a valid RLE file containing a Gosper glider gun, when loaded, then the pattern is correctly placed on the grid and produces gliders
- **AC-3:** Given a Life 1.06 file, when loaded, then the pattern matches the file's cell coordinates
- **AC-4:** Given the glider gun source enabled with 10-second interval, when 10 seconds elapse, then a new glider gun pattern is injected at a random position
- **AC-5:** Given the noise source, when active, then cells flicker on/off following a smooth spatial gradient (not pure random)
- **AC-6:** Given `golf run --pattern glider`, when started, then a glider is placed at the center of the grid
