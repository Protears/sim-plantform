# IW.SIM 总体架构设计

## 1. 架构目标

IW.SIM采用领域驱动、模块化、运行时驱动的工业仿真架构。

核心目标：

- 支撑复杂物流自动化场景建模；
- 支撑PLC/WCS闭环验证；
- 支撑设备行为真实演化；
- 支撑工程项目复用。

## 2. 总体分层

```
应用层
 ├── 场景编辑器
 ├── 虚拟调试平台
 ├── WCS仿真
 └── 运维分析

领域层
 ├── World Model
 ├── Thing Model
 ├── Device Model
 ├── Space Model
 └── Task Model

仿真核心层
 ├── Simulation Runtime
 ├── Event Scheduler
 ├── SimClock
 ├── Behavior Engine
 └── State Pipeline

集成层
 ├── PLC协议
 ├── WCS接口
 ├── OPC UA
 └── REST/SignalR

基础设施层
 ├── PostgreSQL
 ├── EF Core
 ├── OpenTelemetry
 └── Docker
```

## 3. 核心模块职责

### World Model

维护仿真世界中的所有实体以及空间关系。

### Thing Model

提供工业对象统一抽象。

### Simulation Runtime

负责模拟时间推进和对象行为执行。

### Communication Runtime

负责与PLC、WCS以及外部系统通信。

## 4. 设计原则

### 领域优先

业务模型独立于技术实现。

### 行为驱动

设备不是静态数据，而是具有状态和行为的运行实体。

### 确定性仿真

相同输入必须产生一致结果，支持问题复现和自动化测试。
