# IW.SIM Part08 - Communication Architecture

## 1. Overview

Communication Architecture defines integration between IW.SIM and external industrial systems.

## 2. Communication Layers

```text
Application API
      |
Industrial Protocol Layer
      |
Runtime Adapter
      |
Simulation Object
```

## 3. Supported Protocols

- REST API
- SignalR
- S7 Protocol
- OPC UA
- MQTT
- TCP/UDP Custom Protocol

## 4. Communication Management

Features:

- Connection lifecycle
- Message routing
- Protocol conversion
- Monitoring
- Diagnostics

## 5. Industrial Integration

Supports:

- WCS
- PLC
- MES
- Digital Twin Platform
- HIL System

## 6. Design Principle

Keep simulation communication behavior consistent with real industrial systems.