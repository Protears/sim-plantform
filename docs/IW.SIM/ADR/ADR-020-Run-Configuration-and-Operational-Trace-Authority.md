# ADR-020：Run 配置冻结与 Operational Trace 权威

- 状态：Accepted
- 日期：2026-09-19
- 范围：Part03 / Part05 / Part10 / Part11 / Part14

## 背景

前序设计已经确定 Kernel、EventSequence、Recovery、正式 Schema v2 和工程门禁，但仍存在三个实施风险：

1. 运行配置可能在 Run 过程中被局部修改，破坏确定性和恢复可重复性；
2. Event、Telemetry、Audit、Trace 的责任边界不清，导致运维查询从观测数据推断事实；
3. Run 运维操作可能被 API、Worker、CLI 分别实现，状态和幂等规则分叉。

## 决策

1. RunProfile 在 `Frozen` 后成为该 Run 的配置权威；影响确定性结果的字段禁止热更新。
2. Event Store 是业务事实顺序权威，World/Device/Occupancy 是领域状态权威，Operational Audit 是运维动作权威，Telemetry 仅用于观测。
3. `IRunLifecycleCoordinator` 是 Start/Pause/Resume/Recover/Replay/Seal/Archive 的唯一状态转移入口。
4. Trace 通过 `CycleSequence → CommandId → TransferId → OccupancyVersion → ProcessImageVersion → EventSequence` 建立跨模块可追踪链。
5. 缺少关键 Trace 节点、Sequence 缺口或 Audit 写入失败必须显式暴露为 Degraded/Resync/Failed，不得返回“完整成功”。

## 结果

- 配置冻结、运维状态机和审计规则统一；
- API、Worker、CLI 共享 OperationId/ExpectedStateVersion；
- 查询与 SignalR 补发只使用 EventSequence 游标；
- Recovery/Replay 可以生成完整证据链；
- 增加配置、生命周期、Trace、Audit 和发布门禁测试。

## 约束

- Audit 追加写，不允许修改和物理删除；
- Replay 使用新的 ReplayRunId；
- Trace 查询不得从 Telemetry 推断完成事实；
- 任何破坏配置冻结的热更新都必须拒绝并记录 `CFG-005`。
