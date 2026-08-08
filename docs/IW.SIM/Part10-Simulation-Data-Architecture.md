# IW.SIM Part10 - Simulation Data Architecture

## 1. 文档定位

Simulation Data Architecture 是 IW.SIM 仿真平台的数据基础设施设计章节。

数字化仿真系统区别于普通业务系统，其核心数据不是简单业务记录，而是描述工业世界状态变化的数据体系，包括：

- 世界模型数据（World State）
- 设备运行状态数据（Runtime State）
- 控制过程数据（Control State）
- 仿真事件数据（Simulation Event）
- 时间序列数据（Time Series）
- 测试验证数据（Validation Data）

IW.SIM 采用 **Model Driven + Event Driven + Time Driven** 的数据架构，使仿真过程具备确定性、可回放性和可分析能力。

---

# 2. 数据架构总体设计

```
                 Application Layer
                        |
              Simulation Data Service
                        |
 ------------------------------------------------
 |              Data Platform Layer              |
 ------------------------------------------------
 | World Store | Event Store | State Store       |
 | Time Series | Snapshot    | Trace Repository |
 ------------------------------------------------
                        |
              Simulation Kernel
                        |
              Runtime Components
```

数据架构围绕 Simulation Kernel 构建，而不是围绕数据库表设计。

核心原则：

1. 世界状态由模型驱动
2. 状态变化通过事件产生
3. 时间由仿真时钟控制
4. 历史数据支持重放
5. 数据存储服务与运行时解耦

---

# 3. World State 数据模型

World State 描述仿真世界当前状态。

核心对象：

```
World
 ├── Space
 │    ├── Area
 │    ├── Rack
 │    ├── Track
 │    └── Location
 |
 ├── Entity
 │    ├── Device
 │    ├── LoadUnit
 │    ├── Cargo
 │    └── Sensor
 |
 └── Relation
      ├── Containment
      ├── Occupancy
      └── Connectivity
```

World State 不保存设备控制逻辑，只保存工业对象事实状态。

例如：

```
Cargo
{
 Id,
 Position,
 Velocity,
 CurrentLocation,
 Carrier
}
```

---

# 4. Runtime State 数据模型

Runtime State 描述设备执行过程。

包括：

- 当前运行模式
- 当前任务
- 当前动作
- IO状态
- 故障状态
- 能量状态

示例：

```
DeviceRuntimeState
{
 DeviceId,
 Mode,
 RunningTask,
 MotionState,
 AlarmState,
 IOState,
 Timestamp
}
```

Runtime State 生命周期：

```
Created
  |
Initialized
  |
Ready
  |
Running
  |
Paused
  |
Fault
  |
Stopped
```

---

# 5. Event Store 设计

IW.SIM 使用事件作为系统变化的核心记录。

事件结构：

```
SimulationEvent
{
 EventId
 EventType
 SimTime
 Source
 Payload
 CorrelationId
}
```

典型事件：

- CargoCreated
- CargoMoved
- SensorTriggered
- PlcSignalChanged
- DeviceCommandExecuted
- TaskCompleted

事件驱动保证：

- 状态可追踪
- 行为可分析
- 场景可回放

---

# 6. Simulation Time 数据设计

仿真时间与现实时间完全分离。

```
Real Time
    |
Simulation Clock
    |
Event Scheduler
    |
World Update
```

SimTime 包含：

```
SimTime
{
 Tick
 Timestamp
 SpeedRatio
 Mode
}
```

支持：

- 实时运行
- 加速运行
- 单步调试
- 时间跳跃
- 回放模式

---

# 7. Snapshot 快照机制

为了支持大型场景快速恢复，IW.SIM 提供 Snapshot。

快照内容：

- World State
- Runtime State
- Scheduler Queue
- Random Seed
- Simulation Clock

恢复流程：

```
Snapshot Load
      |
Restore World
      |
Restore Runtime
      |
Restore Scheduler
      |
Continue Simulation
```

---

# 8. 数据存储策略

不同数据采用不同存储模型：

|数据类型|存储方式|
|-|-|
|配置模型|关系数据库|
|世界模型|关系数据库+JSON|
|事件日志|Append Only Store|
|实时状态|Memory Cache|
|历史趋势|Time Series Database|
|文件资源|Object Storage|

---

# 9. 数据一致性设计

IW.SIM 不采用数据库事务驱动仿真，而采用仿真事务。

一次 Tick：

```
Input Collection
      |
Runtime Execute
      |
Generate Events
      |
World Update
      |
Persist Snapshot
```

保证：

- 同一输入产生同一结果
- 支持问题复现
- 支持自动化测试

---

# 10. 数据访问架构

提供统一 Data Service：

```
ISimulationDataService

 QueryWorld()
 GetEntity()
 AppendEvent()
 SaveSnapshot()
 QueryHistory()
```

上层模块不直接访问数据库。

---

# 11. 与 Digital Twin Integration 关系

Digital Twin 负责外部映射。

Simulation Data Architecture 负责内部真实状态。

关系：

```
External Twin
      |
Twin Adapter
      |
World Model
      |
Simulation Data
      |
Runtime
```

---

# 12. 工程实现方向

推荐技术：

- PostgreSQL：模型及历史数据
- Redis：运行状态缓存
- Event Log：事件溯源
- OpenTelemetry：数据观测
- Parquet：大规模实验数据分析

---

# 总结

IW.SIM Simulation Data Architecture 是连接 World Model、Runtime、Simulation Kernel 和 Digital Twin 的核心数据基础。

通过事件驱动、时间驱动和模型驱动设计，实现工业仿真系统需要的：

- 可确定执行
- 可回放分析
- 可验证测试
- 可扩展集成
- 可工业化部署
