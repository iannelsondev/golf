# SPEC-007: Network Entropy

**Status:** approved
**Spec Type:** behavior
**Feature Area:** Network Entropy
**Dependencies:** SPEC-001, SPEC-003
**Issue:** #7

## Problem Statement

Network traffic is a rich entropy source — packet sizes, arrival times, DNS query strings, and latency measurements all contain real-world randomness. Mapping network activity to cellular automata state turns the simulation into a live visualization of the machine's network presence.

## Functional Requirements

- **FR-1:** DNS query entropy — listen for DNS resolutions and map query string hashes to cell coordinates, response times to cell age
- **FR-2:** Ping latency mapping — periodically ping configurable targets, map latency values to cell birth patterns (high latency → more chaos)
- **FR-3:** Socket data entropy — open a listening UDP/TCP socket, hash incoming byte payloads to cell coordinates
- **FR-4:** Network interface stats — read bytes/packets counters from `/sys/class/net/*/statistics/`, map rate-of-change to entropy density
- **FR-5:** Configurable target list for ping and DNS sources — defaults to well-known public DNS servers
- **FR-6:** Graceful degradation — if network is unavailable or permissions insufficient, source reports unavailable

## Non-Functional Requirements

- **NFR-1:** Network operations must not block the simulation thread — all I/O is async via tokio
- **NFR-2:** Ping intervals configurable, default 1 second, minimum 100ms

## Acceptance Criteria

- **AC-1:** Given active DNS resolution on the system, when the DNS source is running, then cell coordinates derived from query hashes appear on the grid
- **AC-2:** Given ping targets with varying latency, when pings complete, then higher latency targets produce more cell births
- **AC-3:** Given incoming UDP packets on the listening socket, when received, then packet data is hashed to cell coordinates
- **AC-4:** Given network interface stats showing high throughput, when polled, then entropy density increases proportionally
- **AC-5:** Given no network access, when the network source is requested, then it reports unavailable without crashing
- **AC-6:** Given the ping source running, when a target becomes unreachable, then it is skipped gracefully and other targets continue
