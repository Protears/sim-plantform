# IW.SIM Observability Architecture

## Overview
IW.SIM adopts unified observability for simulation runtime, engineering services and integration components.

## Three Pillars

### Logging
Structured logs based on Serilog with trace correlation.

### Metrics
Runtime indicators include simulation tick latency, device count, event throughput and communication status.

### Tracing
OpenTelemetry provides distributed tracing across Web API, simulation engine and external integrations.

## Technology Stack

- OpenTelemetry
- OTLP
- Prometheus
- Grafana
- Loki

## Goals

Provide diagnosis capability for long-running industrial simulation scenarios.
