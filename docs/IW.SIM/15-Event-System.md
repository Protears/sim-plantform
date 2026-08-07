# Event System Design

## Overview
IW.SIM uses event driven architecture as the foundation of deterministic simulation.

## Core Components

- Event Queue
- Simulation Clock
- Scheduler
- Event Dispatcher
- Domain Events

## Event Lifecycle

Create -> Schedule -> Execute -> Publish -> Observe

## Applications

- Device state change
- Sensor trigger
- Cargo movement
- PLC signal update
- Simulation control

## Benefits

Provides scalability, traceability and replay capability.
