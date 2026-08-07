# IW.SIM Part25 - Industrial Protocol Integration

## Overview

支持工业现场协议仿真和联调。

## Protocols

- Siemens S7
- OPC UA
- MQTT
- TCP/UDP
- Custom Protocol

## Architecture

```text
Protocol Adapter
 -> Runtime
 -> World Model
 -> Simulation Kernel
```

## Extension

通过 Plugin Architecture 扩展协议能力。