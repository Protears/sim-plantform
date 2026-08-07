# ADR-006 Digital Twin Model

## Status
Accepted

## Context
IW.SIM needs a unified representation connecting physical equipment, simulation entities and operational data.

## Decision
Adopt a layered digital twin model:

- Physical Layer: real equipment and sensors
- Model Layer: industrial thing model and geometry
- Runtime Layer: simulation entities and behaviors
- Data Layer: events, telemetry and history

## Consequences
Provides consistent mapping between real systems and simulation runtime.