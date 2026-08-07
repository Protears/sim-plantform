# IW.SIM Part12 - Data Architecture

## 1. Overview

IW.SIM data architecture separates configuration data, runtime state and event history.

## 2. Data Layers

```text
Configuration Data
        |
Domain Model
        |
Runtime State
        |
Event Log
```

## 3. Storage

- PostgreSQL production database
- EF Core persistence
- Event log storage
- Time series telemetry

## 4. Design Principles

- Domain first
- Event driven
- Append only history
- Query optimized projection

## 5. Observability

Integrated with OpenTelemetry, Prometheus and Grafana.