# IW.SIM Part09 - Digital Twin Integration

## 1. Overview

IW.SIM Digital Twin Integration provides synchronization between simulation world and physical industrial systems.

## 2. Architecture

```text
Physical System
      |
IoT / PLC / OPC UA
      |
Twin Gateway
      |
World Model
      |
Simulation Kernel
```

## 3. Core Functions

- Runtime state synchronization
- Equipment status mapping
- Asset model binding
- Historical data integration
- Simulation result feedback

## 4. Data Model

Digital Twin consists of:

- Asset
- Thing Model
- Runtime State
- Telemetry
- Events
- Commands

## 5. Integration Protocols

Support:

- OPC UA
- MQTT
- REST API
- SignalR
- Industrial TCP protocols

## 6. Implementation

Based on .NET 10, event driven architecture and domain model.