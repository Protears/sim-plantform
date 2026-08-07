# Communication Architecture

## Overview
IW.SIM communication architecture provides unified connectivity between simulation runtime and external systems.

## Communication Channels

- REST API
- SignalR realtime channel
- PLC protocol adapters
- Industrial Ethernet protocols

## Design Principles

- Loose coupling
- Protocol abstraction
- High reliability
- Observable communication

## Adapter Pattern

External protocols are converted into internal domain commands and events.

## Supported Targets

PLC, WCS, MES, SCADA and engineering tools.
