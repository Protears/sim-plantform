# IW.SIM Part10 - HIL Simulation

## 1. Overview

Hardware In Loop enables physical controllers and virtual equipment to operate together.

## 2. Architecture

```text
PLC Hardware
     |
Communication Layer
     |
Simulation Runtime
     |
Virtual Device Model
```

## 3. Components

- Virtual Plant
- PLC Interface
- IO Simulator
- Timing Controller
- Test Scenario Engine

## 4. Features

- PLC program verification
- Device commissioning simulation
- Fault injection
- Automated regression testing

## 5. Timing Model

Support real-time clock synchronization and deterministic simulation.