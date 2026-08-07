# Data Architecture

## Overview
IW.SIM data architecture separates runtime state, engineering data and historical records.

## Data Categories

- Project configuration
- Domain model data
- Simulation runtime state
- Event logs
- Monitoring data

## Storage Design

Production database uses PostgreSQL with EF Core based domain persistence.

## Principles

- Domain first
- Append only event records
- Traceable simulation history
- Version controlled models

## Query Model

CQRS style read models support visualization and analysis scenarios.
