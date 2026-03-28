# SPEC-003: Entropy Trait

**Status:** approved
**Spec Type:** types
**Feature Area:** Entropy Trait
**Dependencies:** SPEC-001
**Issue:** #3

## Problem Statement

The framework's core differentiator is pluggable entropy — the ability to feed the simulation from any data source in real time. This spec defines the `EntropySource` trait, the `EntropyEvent` types, and the channel-based event delivery mechanism. Every entropy source (GPU, audio, network, AI, files) implements this trait.

## Functional Requirements

- **FR-1:** `EntropySource` trait with `name()`, `start()`, `stop()`, and event stream methods
- **FR-2:** `EntropyEvent` enum covering: `SetCells(Vec<(x, y)>)`, `ClearRegion(Rect)`, `ApplyPattern(Pattern, x, y)`, `MutateRules(RuleSet)`, `EnergyPulse(center, radius, intensity)`
- **FR-3:** Async channel-based delivery — sources push `EntropyEvent`s into an `mpsc` channel, simulation engine drains between steps
- **FR-4:** Source registry — register, list, enable/disable sources at runtime
- **FR-5:** Event rate limiting — configurable max events per step per source to prevent any single source from overwhelming the simulation
- **FR-6:** Source metadata — each source reports its type, status (running/stopped/error), and event rate for the stats overlay

## Non-Functional Requirements

- **NFR-1:** Channel drain must complete in under 100 microseconds for up to 100 pending events
- **NFR-2:** Adding or removing a source must not pause the simulation

## Acceptance Criteria

- **AC-1:** Given a struct implementing `EntropySource`, when registered with the source registry, then it appears in the source list and can be started
- **AC-2:** Given a running source producing `SetCells` events, when the simulation drains the channel, then those cells are set before the next step
- **AC-3:** Given a source producing 1000 events/step with a rate limit of 50, when drained, then only 50 events are applied and the rest are discarded
- **AC-4:** Given two sources running simultaneously, when both push events, then events from both are interleaved in the channel
- **AC-5:** Given a running source, when `stop()` is called, then no further events are produced and the source status is `stopped`
- **AC-6:** Given an `EnergyPulse` event, when applied, then cells within the radius are set alive with intensity-scaled probability
