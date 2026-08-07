# IW.SIM Device实例模型

## 1. 概述

Device是Thing模型在具体工业设备上的运行实例。

它连接物模型定义和实际仿真运行。

## 2. Device结构

```text
Device
 ├── DeviceIdentity
 ├── DeviceType
 ├── Capabilities
 ├── Runtime
 ├── State
 ├── IO Mapping
 └── Communication Adapter
```

## 3. 示例

Conveyor Device：

- Motor Capability
- Sensor Capability
- Transport Capability

RGV Device：

- Move Capability
- Position Capability
- PLC Interface

## 4. 设计原则

设备实例只保存运行状态和配置，行为逻辑由Runtime执行，实现模型复用。