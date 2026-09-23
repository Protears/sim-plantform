# ADR-024：TaskCompleted 必须由 Occupancy Commit 授权

## 状态
Accepted

## 背景

WCS、PLC、Device Runtime、Telemetry 都可能观察到“设备动作完成”，但只有 World/Occupancy 才能确认货载已经从源位置转移到目标位置。若由设备完成信号直接发布 TaskCompleted，会出现货载未提交、目标位置冲突、重复完成和 Replay 不一致。

## 决策

1. `TaskCompleted` 只能在 T-COMPLETE 事务中产生。
2. T-COMPLETE 必须同时满足：DeviceExecutionSucceeded、OccupancyTransferCommitted、RequiredFeedbackPublished。
3. Task、Transfer、Occupancy、Event、Outbox 在同一事务边界内更新。
4. WCS/PLC/Telemetry 只能提交意图或观测，不能写完成事实。
5. Recovery/Replay 只重放已提交 TaskCompleted，不从 Telemetry 推断。

## 后果

- 编排状态晚于设备动作一个事实提交边界，这是有意设计。
- 需要 Task 与 Transfer 使用统一 CorrelationId。
- 需要补充 T-COMPLETE 失败重试和补偿测试。
- 查询层必须区分 `DeviceExecutionSucceeded` 与 `TaskCompleted`。

## 约束

- 任何绕过 OccupancyAuthority 的完成事件视为架构违规。
- `TaskCompleted` 必须携带 `TransferId、OccupancyVersion、StateHash、EventSequence`。
