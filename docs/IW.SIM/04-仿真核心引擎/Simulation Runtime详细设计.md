# IW.SIM Simulation Runtime详细设计

## 1. 定位

Simulation Runtime 是 IW.SIM运行时核心，负责驱动整个虚拟工业世界持续运行。

## 2. 核心职责

- 管理仿真生命周期；
- 调度设备行为；
- 执行事件；
- 维护世界状态；
- 发布运行观察数据。

## 3. Runtime结构

```
SimulationRuntime
 ├── SimClock
 ├── EventScheduler
 ├── WorldContext
 ├── BehaviorExecutor
 ├── DeviceRuntime
 └── ObservationPipeline
```

## 4. 执行周期

每个仿真周期：

1. 推进仿真时间；
2. 获取到期事件；
3. 执行业务行为；
4. 更新实体状态；
5. 输出观察结果。

## 5. 工程价值

Runtime将设备模型、控制逻辑和世界模型连接起来，是虚拟调试的核心基础。