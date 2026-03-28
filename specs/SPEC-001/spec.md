# SPEC-001: Core Engine

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Core Engine
**Dependencies:** none
**Issue:** #1

## Problem Statement

The Game of Life framework needs an N-dimensional simulation engine rooted in set theory. Cells exist as elements of an N-dimensional discrete space. Neighborhoods are defined as set operations — Moore (all adjacent cells in N dimensions) or von Neumann (axis-aligned neighbors only). Rules are predicates over neighborhood cardinality: Conway's B3/S23 is `birth: {3}, survival: {2, 3}` in the 2D Moore case, but the same formalism extends to 3D, 4D, and beyond. 2D is the default, not a hard limit. This is the foundation everything else depends on.

## Functional Requirements

- **FR-1:** `NdGrid<const N: usize>` — generic N-dimensional grid backed by a flat buffer, indexed by `[usize; N]` coordinates, with configurable extents per dimension
- **FR-2:** Set-theoretic rule system — rules defined as `RuleSet { birth: BTreeSet<u32>, survival: BTreeSet<u32>, neighborhood: NeighborhoodKind }` where `NeighborhoodKind` is `Moore` or `VonNeumann` in N dimensions
- **FR-3:** Neighbor counting generalized to N dimensions — Moore neighborhood has `3^N - 1` neighbors, von Neumann has `2N` neighbors
- **FR-4:** Configurable edge behavior — toroidal (wrapping) and bounded (dead border) in each dimension independently
- **FR-5:** Cell state tracking — alive/dead plus cell age (generation count since birth) for rendering heatmaps
- **FR-6:** Accept `EntropyEvent` injections between steps — set cells, clear regions, apply patterns at N-dimensional coordinates, mutate rules
- **FR-7:** Named rule presets — `conway` (B3/S23 2D Moore), `highlife` (B36/S23), `day-night` (B3678/S34678), `3d-life` (B5/S45 3D Moore), and custom

## Non-Functional Requirements

- **NFR-1:** Step a 200x50 2D grid (10,000 cells) in under 1ms
- **NFR-2:** Zero heap allocations during steady-state stepping (double-buffer swap)
- **NFR-3:** 3D grids up to 50x50x50 (125,000 cells) must step in under 50ms

## Acceptance Criteria

- **AC-1:** Given a 2D grid with B3/S23 rules, when a glider pattern is placed and stepped 4 times, then it translates one cell diagonally
- **AC-2:** Given a toroidal 2D grid, when a glider reaches the right edge, then it wraps to the left edge
- **AC-3:** Given a 3D grid with B5/S45 Moore rules, when stepped, then neighbor counts correctly include all 26 adjacent cells
- **AC-4:** Given cells of varying ages, when stepped, then surviving cells increment age and newborn cells start at age 1
- **AC-5:** Given an `EntropyEvent::SetCells` with N-dimensional coordinates, when injected, then those cells are alive in the next step's input
- **AC-6:** Given a custom `RuleSet { birth: {3,6}, survival: {2,3} }`, when the HighLife replicator is placed and stepped, then it replicates correctly
- **AC-7:** Given a 200x50 2D grid, when benchmarked over 1000 steps, then average step time is under 1ms

## Constraints

- Double-buffer strategy: read from buffer A, write to buffer B, swap
- No external dependencies for core grid logic — pure Rust, no unsafe in hot path
- `NdGrid` parameterized by `const N: usize` — dimensions are a compile-time constant for performance
