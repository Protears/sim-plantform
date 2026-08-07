# IW.SIM Part02 - World Model

## 1. Overview

World Model 是 IW.SIM 工业仿真平台的核心领域模型，用于描述工业系统中的空间、设备、载荷、资源以及运行状态。

World Model 不直接执行控制逻辑，而负责提供统一的工业世界语义，为 Simulation Kernel、Device Runtime、PLC Runtime 和上层应用提供一致的数据基础。

## 2. Design Goals

- 统一描述物流自动化系统中的实体对象
- 支持静态布局与动态运行状态分离
- 支持离散事件仿真和实时仿真
- 支持数字孪生数据映射
- 支持 PLC/WCS/HIL 联调场景

## 3. Core Concepts

### 3.1 World

World 表示一个完整工业仿真空间，例如：

- 自动化立库
- 输送系统
- 机器人工作站
- 生产线

World 包含多个 WorldEntity。

### 3.2 WorldEntity

所有工业对象统一抽象为实体：

```text
WorldEntity
 ├── Identity
 ├── Geometry
 ├── State
 ├── Properties
 ├── Behaviors
 └── Events
```

### 3.3 Spatial Model

空间模型负责描述工业设备的位置关系。

核心对象：

- Site
- Area
- Aisle
- Rack
- Track
- Segment
- Position

支持二维布局和三维空间扩展。

## 4. Industrial Object Model

### LoadUnit

表示被搬运对象：

```text
LoadUnit
 ├── Id
 ├── Cargo
 ├── Size
 ├── Weight
 ├── CurrentPosition
 └── State
```

### Device

表示工业设备：

```text
Device
 ├── DeviceType
 ├── Capabilities
 ├── RuntimeBinding
 ├── IO Mapping
 └── OperationalState
```

设备类型包括：

- Conveyor
- Stacker Crane
- RGV
- AGV
- Robot

## 5. State Model

对象状态分为：

### Static State

设计阶段确定：

- 几何尺寸
- 安装位置
- 能力参数

### Dynamic State

运行期间变化：

- 位置
- 速度
- 占用状态
- 故障状态
- 执行任务

## 6. Event Driven Model

World Model 通过事件驱动保持动态一致性。

主要事件：

- EntityCreated
- PositionChanged
- CargoMoved
- SensorTriggered
- DeviceFaulted

Simulation Kernel 消费事件并推进仿真时间。

## 7. Domain Relationship

```text
World
 ├── Areas
 │    └── Devices
 │          └── Runtime
 │
 ├── SpatialGraph
 │
 └── LoadUnits
```

## 8. Extension Capability

World Model 支持：

- ROS2 Robot Model Integration
- CAD/BIM Layout Import
- Digital Twin Synchronization
- PLC Address Mapping
- HIL Hardware Mapping

## 9. Implementation Direction

技术实现采用领域驱动设计：

- POCO Domain Model
- Clean Architecture
- Event Sourcing Compatible
- EF Core Persistence
- Signal/Event Streaming

World Model 是 IW.SIM 构建工业级仿真能力的基础层。