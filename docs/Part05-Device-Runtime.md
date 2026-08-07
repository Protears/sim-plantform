# IW.SIM Part05 - Device Runtime

## 1. Overview

Device Runtime 是 IW.SIM 工业仿真平台中负责设备行为执行的核心运行时层。

它将 World Model 中的设备实体转换为可执行的工业设备模型，实现设备状态管理、控制命令执行、协议适配以及仿真行为驱动。

Device Runtime 位于 Simulation Kernel 与现场控制系统之间，是连接工业语义模型和设备执行模型的关键组件。

## 2. Architecture

```text
Device Runtime

+-----------------------------+
| Device Lifecycle Manager    |
+-----------------------------+
| Device State Machine        |
+-----------------------------+
| Capability Runtime          |
+-----------------------------+
| Command Execution Pipeline  |
+-----------------------------+
| Driver / Protocol Adapter   |
+-----------------------------+
| Simulation Kernel Interface |
+-----------------------------+
```

## 3. Device Lifecycle

设备生命周期包括：

- Created
- Initialized
- Starting
- Running
- Stopping
- Stopped
- Faulted
- Disabled

Runtime 负责生命周期状态转换和事件通知。

## 4. Device State Machine

设备状态模型：

```text
Idle
 |
 v
Ready
 |
 v
Executing
 |
 +----> Completed
 |
 +----> Fault
```

状态变化通过领域事件发布。

## 5. Capability Runtime

设备能力通过 Capability 抽象：

例如输送机：

```text
ConveyorCapability
 ├── Start
 ├── Stop
 ├── SetSpeed
 ├── Reverse
 └── SensorInput
```

堆垛机：

```text
StackerCraneCapability
 ├── MoveHorizontal
 ├── MoveVertical
 ├── ForkExtend
 └── ForkRetract
```

## 6. Command Pipeline

设备控制流程：

```text
Controller Command
        |
        v
Command Validation
        |
        v
Capability Runtime
        |
        v
Motion / Logic Execution
        |
        v
State Update
        |
        v
Event Publish
```

## 7. Driver Architecture

Driver 层负责工业协议和品牌适配。

支持：

- Siemens S7
- OPC UA
- Modbus TCP
- MQTT
- Custom TCP Protocol

结构：

```text
Device Runtime
      |
Driver Interface
      |
Protocol Adapter
      |
Physical Controller
```

## 8. Motion Runtime

运动设备支持：

- Position Control
- Velocity Control
- Acceleration Model
- Collision Detection
- Occupancy Management

典型设备：

- Conveyor
- RGV
- AGV
- Stacker Crane
- Robot

## 9. Runtime Integration

与其他模块关系：

```text
World Model
      |
      v
Device Runtime
      |
      +---- Simulation Kernel
      |
      +---- PLC Runtime
      |
      +---- Communication Runtime
```

## 10. Real-time Simulation

Device Runtime 支持：

- 离散事件驱动
- 周期 Tick 更新
- 实时设备协议交互
- HIL 硬件联调

## 11. .NET Implementation

推荐实现：

- .NET 10
- C# 13
- Generic Host
- Dependency Injection
- BackgroundService
- Channel<T>
- OpenTelemetry

## 12. Summary

Device Runtime 是 IW.SIM 实现工业设备级仿真的核心执行层，通过统一运行时模型，实现设备行为仿真、协议适配以及控制系统联调。