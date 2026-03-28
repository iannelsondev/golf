# SPEC-005: GPU Entropy

**Status:** approved
**Spec Type:** behavior
**Feature Area:** GPU Entropy
**Dependencies:** SPEC-001, SPEC-003
**Issue:** #5

## Problem Statement

GPU VRAM contains raw entropy — partially overwritten framebuffers, shader intermediate values, uninitialized memory. Reading this data and mapping it to cellular automata state creates a direct bridge between the GPU's computational residue and the simulation. This is the flagship entropy source that makes golf unique.

## Functional Requirements

- **FR-1:** Enumerate available CUDA devices and report device name, VRAM total/free, compute capability
- **FR-2:** Read arbitrary VRAM regions via CUDA device memory access — configurable offset, size, and read interval
- **FR-3:** Bit-to-cell mapping strategies: threshold (byte > N → alive), bitfield (each bit = one cell), entropy density (hash bytes into probability)
- **FR-4:** Configurable entropy injection rate — how many cells per read, how frequently reads occur
- **FR-5:** VRAM region selection modes: random walk (different region each read), fixed (same region, watch it change), sequential scan (sweep through VRAM)
- **FR-6:** Graceful degradation — if no CUDA runtime or no GPU, source reports unavailable and the simulation continues without it

## Non-Functional Requirements

- **NFR-1:** VRAM read must complete in under 5ms for a 64KB region
- **NFR-2:** Must not interfere with running GPU workloads — read-only access to device memory

## Acceptance Criteria

- **AC-1:** Given a system with a CUDA GPU, when the GPU source is started, then device info (name, VRAM) is reported
- **AC-2:** Given a 64KB VRAM read, when mapped with threshold strategy, then cells are set alive where bytes exceed the threshold
- **AC-3:** Given the random walk mode, when two consecutive reads occur, then they target different VRAM offsets
- **AC-4:** Given a system without CUDA, when the GPU source is requested, then it reports unavailable without crashing
- **AC-5:** Given an entropy injection rate of 100 cells/read at 10 reads/sec, when running, then approximately 1000 cells/sec are injected
- **AC-6:** Given a running GPU workload, when VRAM reads occur simultaneously, then the GPU workload is not measurably impacted
