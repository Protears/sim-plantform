# IW.SIM API Design

## Principles

REST API and realtime event APIs are provided for engineering applications.

## API Style

- URL versioning: /v1 /v2
- JSON camelCase
- Domain oriented resources

## Interfaces

### Project API
Manage simulation projects and configurations.

### Model API
Access device models and world models.

### Simulation API
Control simulation lifecycle.

### Event API
Subscribe simulation runtime events.

## Realtime Communication

SignalR provides runtime status updates and visualization synchronization.
