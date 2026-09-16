# Part 14：Slice D 工程任务切片——Kernel 与 PLC/设备闭环

## 1. 目标

将前序契约拆分为可独立开发、可独立验收的任务切片 D，范围为：Kernel 排序、异步输入背压、Snapshot 一致性、PLC—设备反馈追踪。

## 2. 任务包

| 任务 | 产物 | 前置 | 验收 |
|---|---|---|---|
| D-01 | `ISimulationCommandQueue` | Part03 | 同 Tick 固定排序 |
| D-02 | `IInputIngress` + BackpressurePolicy | Part08/12 | 满载策略和指标 |
| D-03 | Snapshot Coordinator | Part10 | Capture/Restore Hash 一致 |
| D-04 | PLC Trace DTO/Event | Part06 | Command→Feedback 可反查 |
| D-05 | Architecture/Contract Tests | Part14 | CI 失败即阻断 |

## 3. Definition of Done

- 领域和 Kernel 项目无基础设施引用。
- 所有公共 DTO 含 `RunId/CorrelationId/SchemaVersion`。
- 每个任务包含正常、冲突、超时、恢复至少 1 个测试。
- 每个任务有指标：延迟、拒绝、重复、恢复耗时。
- 任务完成后更新 Part10/11/12 关联契约，不允许只更新实现文档。

## 4. 发布门禁

Slice D 进入联调分支必须同时通过：

1. 排序确定性测试。
2. 100 个并发设备输入压测。
3. Snapshot 恢复和 Replay Hash 测试。
4. PLC Feedback 旧版本拒绝测试。
5. Outbox/SignalR 断线补发测试。
6. 72 小时稳定性测试。

任一门禁失败，版本状态为 `Blocked`，不得进入 HIL 联调。