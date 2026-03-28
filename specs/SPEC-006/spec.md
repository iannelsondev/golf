# SPEC-006: Audio Entropy

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Audio Entropy
**Dependencies:** SPEC-001, SPEC-003
**Issue:** #6

## Problem Statement

Sound is entropy. Microphone input, system audio, or music playing on the machine can drive the simulation — FFT spectral analysis maps frequency bands to grid regions, beat detection triggers cell pulses, and amplitude modulates birth probability. The simulation becomes a real-time audio visualizer that also happens to be a cellular automaton.

## Functional Requirements

- **FR-1:** Capture audio from the default input device (microphone) or a specified device via cpal
- **FR-2:** Real-time FFT spectral analysis — decompose audio into frequency bands using rustfft
- **FR-3:** Frequency-to-grid mapping — map low/mid/high frequency bands to different grid regions or dimensions (e.g., bass → bottom rows, treble → top rows)
- **FR-4:** Beat detection — detect percussive transients and emit `EnergyPulse` events that radiate outward from a point
- **FR-5:** Amplitude-modulated birth probability — louder audio increases the chance of spontaneous cell birth across the grid
- **FR-6:** Graceful degradation — if no audio device available, source reports unavailable without crashing

## Non-Functional Requirements

- **NFR-1:** Audio processing latency under 20ms from capture to entropy event emission
- **NFR-2:** FFT window size configurable (256, 512, 1024, 2048 samples) for frequency resolution vs latency tradeoff

## Acceptance Criteria

- **AC-1:** Given a microphone capturing audio, when a loud sound occurs, then cell birth probability increases in the next simulation step
- **AC-2:** Given audio with strong bass, when FFT maps frequency bands to grid regions, then the bottom rows show more activity
- **AC-3:** Given a percussive beat, when detected, then an `EnergyPulse` event is emitted with center coordinates and radius
- **AC-4:** Given silence, when the audio source is active, then minimal or no entropy events are produced (not random noise)
- **AC-5:** Given no audio device available, when the audio source is requested, then it reports unavailable without crashing
- **AC-6:** Given audio processing, when benchmarked, then capture-to-event latency is under 20ms
