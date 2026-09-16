# ADR-017：Kernel 是唯一 SimTick 权威

## 状态

Accepted

## 背景

PLC、HIL、通信和后台持久化均可能拥有自己的时钟或线程。若这些模块直接推进仿真时间，会造成同一 Run 的设备状态、Occupancy、事件序列不可重复。

## 决策

1. `SimulationKernel` 是唯一推进 `SimTick` 的组件。
2. PLC/HIL/Communication 只能提交带 `TargetSimTick` 的输入。
3. WallClock 只用于连接超时、指标和调度唤醒，不参与领域结果计算。
4. Snapshot、Replay、Event Store 均以 Kernel 的 `SimTick` 和 Sequence 为事实基准。
5. 任何绕过 Kernel 修改 World State 的实现视为架构违规。

## 后果

- 异步输入必须进入有界队列和 TickBoundary。
- 低延迟不能通过“直接改状态”实现，只能优化 InputIngress 和调度预算。
- Replay 可脱离原始网络和线程时序。

## 强制验证

- 架构测试禁止 HIL/Communication 调用 `AdvanceTick`。
- 单元测试验证同输入集合在不同到达顺序下 Hash 相同。
- 运行时诊断记录 `SimTick/Phase/SourceId/LocalSequence`。