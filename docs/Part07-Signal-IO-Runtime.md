# IW.SIM Part07 - Signal IO Runtime

## 1. Overview

Signal IO Runtime abstracts industrial signals between PLC, devices and simulation models.

## 2. Signal Model

```text
Signal
 ├── Address
 ├── Type
 ├── Value
 ├── Quality
 └── Timestamp
```

## 3. Functions

- IO mapping
- Signal routing
- State synchronization
- Event triggering
- Protocol adaptation

## 4. Supported Signals

- Sensor
- Actuator
- Motion command
- Safety signal
- Alarm signal

## 5. Mapping Architecture

```text
PLC Address
    ↓
Signal Mapping
    ↓
Thing Capability
    ↓
Device Runtime
```

## 6. Design Goals

Provide consistent industrial IO abstraction for simulation and real equipment.