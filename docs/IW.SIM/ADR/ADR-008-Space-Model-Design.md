# ADR-008 Space Model Design

## Status
Accepted

## Context
Industrial simulation requires unified spatial representation.

## Decision
Use hierarchical space modeling:

Space -> Area -> Aisle -> Rack -> Place

Each entity contains geometry, topology and operational constraints.

## Consequences
Supports visualization, routing, collision detection and simulation.