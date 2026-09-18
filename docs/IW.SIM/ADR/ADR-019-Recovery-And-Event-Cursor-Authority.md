# ADR-019：Recovery 与 EventSequence 游标权威

## 状态

Accepted

## 背景

前序文档分别定义了 Snapshot、Event Store、SignalR、Replay 和 PLC Feedback，但若各模块自行选择时间戳、数据库自增 ID 或本地计数作为恢复游标，会导致断线补发、Replay 和恢复后的事件顺序不可证明。

## 决策

1. `EventSequence` 是同一 Run 内已提交事实的唯一顺序游标，由 EventStore 分配。
2. Snapshot 必须记录 `BaseEventSequence`、`InputCursor`、`SimTick`、`ClockEpoch`、`StateHash`。
3. Run Event Query、SignalR 补发、Projection Checkpoint、Replay From/To 都使用 EventSequence。
4. Recovery 只允许从 Committed Snapshot 或 Full Replay 进入；禁止从 Telemetry、SignalR 或 Projection 反推事实。
5. Replay 使用新的 `ReplayRunId`，只读源 Run 事实，不写回源 Run。
6. Recovery/Replaying 期间禁止发布对外完成事件，直到 Hash 校验通过并显式 Resume。

## 影响

- API、SignalR、Projection 和测试必须统一游标语义。
- 数据库需要唯一索引 `(run_id, event_sequence)`。
- 客户端必须保存最后确认的 EventSequence，并支持 ResyncRequired。
- 运维排障可以使用 `RunId + EventSequence` 重建完整链路。

## 验证

- 断线后从最后确认游标补发，不能重复或跳过。
- 相同 Snapshot 重复恢复结果 Hash 一致。
- 不同网络到达顺序的 Replay 结果 Hash 一致。
- EventSequence 缺口、Hash 不匹配、Schema 不兼容均不得进入 Running。