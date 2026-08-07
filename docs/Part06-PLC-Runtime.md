# IW.SIM Part06 - PLC Runtime

## 1. Overview

PLC Runtime provides virtual PLC execution capability for IW.SIM, enabling PLC logic, I/O process and device simulation integration.

## 2. Architecture

```text
PLC Runtime
 ├── Scan Cycle Engine
 ├── Program Executor
 ├── IO Image Table
 ├── Signal Mapper
 ├── Communication Adapter
 └── Diagnostic Service
```

## 3. Scan Cycle Model

PLC execution follows industrial scan cycle:

1. Read Input Image
2. Execute Logic Program
3. Update Output Image
4. Exchange Runtime Events

## 4. IO Model

Supports:

- Digital Input
- Digital Output
- Analog Input
- Analog Output
- PLC Data Block
- Field Bus Signal

## 5. Siemens Integration

Supports virtual Siemens S7 environment:

- S7 Communication
- DB Mapping
- Snap7 Compatibility
- TIA Portal Data Import Extension

## 6. Synchronization

PLC Runtime synchronizes with:

- Simulation Kernel
- Device Runtime
- WCS Controller

## 7. Implementation

Based on .NET 10 and event-driven architecture.