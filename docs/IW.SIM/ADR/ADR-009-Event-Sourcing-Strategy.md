# ADR-009 Event Sourcing Strategy

## Status
Accepted

## Decision
Adopt append-only event records for simulation history.

## Goals
- Deterministic replay
- Debugging
- Traceability
- Historical analysis

## Implementation
Simulation events are persisted with timestamps, entity identity and event payload.