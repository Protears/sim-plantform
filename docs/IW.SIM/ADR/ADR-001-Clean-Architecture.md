# ADR-001 Clean Architecture

## Status
Accepted

## Context
IW.SIM requires long lifecycle maintenance and complex industrial domain evolution.

## Decision
Adopt Clean Architecture with clear dependency direction.

Layers:
- Domain
- Application
- Infrastructure
- Presentation

## Consequences
Business models remain independent from frameworks and infrastructure.
