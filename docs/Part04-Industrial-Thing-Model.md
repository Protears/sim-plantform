# IW.SIM Part04 - Industrial Thing Model

## 1. Overview

Industrial Thing Model 是 IW.SIM 平台描述工业对象语义的核心模型层，用于统一表达设备、组件、资源和工业能力。

Thing Model 位于 World Model 与 Runtime Layer 之间，为仿真对象、PLC 控制、数字孪生以及工业协议提供统一语义。

## 2. Design Objectives

设计目标：

- 统一工业对象描述
- 分离对象定义与运行实例
- 支持多协议映射
- 支持设备能力扩展
- 支持数字孪生同步
- 支持模型版本演进

## 3. Thing Model Core Structure

```text
Thing
 ├── Identity
 ├── Metadata
 ├── Properties
 ├── Capabilities
 ├── Services
 ├── Events
 └── RuntimeBinding
```

## 4. Thing Definition

Thing 表示一个工业对象类型定义。

例如：

- Conveyor
- Stacker Crane
- RGV
- PLC Controller
- Sensor

Thing Definition 描述对象具备什么能力，而不是当前运行状态。

## 5. Property Model

Property 描述工业对象的数据属性。

示例：输送机

```text
Conveyor
 ├── Length
 ├── Speed
 ├── Acceleration
 ├── Direction
 ├── LoadCapacity
 └── RunningState
```

属性支持：

- 数据类型
- 单位
- 访问权限
- 默认值
- 数据质量

## 6. Capability Model

Capability 表示工业对象提供的能力。

例如：

```text
Conveyor Capability
 ├── Move
 ├── Stop
 ├── Reverse
 └── ResetFault
```

Capability 是 Runtime 调用设备行为的统一入口。

## 7. Service Model

Service 描述可执行操作。

```text
Service
 ├── Name
 ├── Input Parameters
 ├── Output Result
 └── Execution Policy
```

例如：

- StartMotor
- MoveToPosition
- LoadCargo
- UnloadCargo

## 8. Event Model

Event 描述工业对象产生的状态变化。

典型事件：

- SensorTriggered
- CargoArrived
- DeviceStarted
- FaultOccurred
- PositionChanged

事件进入 Simulation Kernel 的事件流。

## 9. PLC Signal Mapping

Thing Model 支持 PLC I/O 映射。

```text
Thing Property
        |
        v
Signal Mapping
        |
        v
PLC Address
```

示例：

```text
Conveyor.Running
    -> DB100.DBX0.0

Conveyor.Speed
    -> DB100.DBD10
```

支持：

- Bit Mapping
- Byte Mapping
- Struct Mapping
- Field Path Mapping

## 10. Runtime Binding

Thing Instance 与运行环境绑定：

```text
Thing Model
      |
      v
Thing Instance
      |
      v
Device Runtime
```

Runtime 负责：

- 协议通信
- 状态刷新
- 指令执行

## 11. Device Model vs Thing Model

区别：

| Model | Responsibility |
|---|---|
| Thing Model | 描述工业语义 |
| Device Model | 描述设备实例 |
| Runtime Model | 描述执行环境 |

三者共同组成工业对象运行体系。

## 12. Version Management

Thing Model 支持版本管理：

```text
ThingModel
 ├── Version
 ├── Schema
 ├── Compatibility
 └── Migration
```

支持模型升级和历史仿真复现。

## 13. External Integration

支持：

- OPC UA Information Model
- MQTT Topic Model
- AutomationML
- ROS2 Interface
- Digital Twin Platform

## 14. Implementation Direction

.NET 实现方向：

- POCO Domain Entity
- JSON Schema Validation
- EF Core Persistence
- Event Driven Runtime Binding
- Source Generator 优化模型访问

Industrial Thing Model 是 IW.SIM 实现工业语义统一和跨协议融合的基础。