# Part 14：World State Reconciliation Test Matrix

| ID | 场景 | 期望 |
|---|---|---|
| WR-001 | 旧 ProcessImageVersion | 拒绝覆盖，记录 D2 |
| WR-002 | OccupancyVersion 冲突 | 返回 WORLD-OCC-409 |
| WR-003 | Device 完成但 Transfer 未 Commit | 不发布 Completed |
| WR-004 | LayoutHash 不一致 | Run 进入 Failed |
| WR-005 | BindingSnapshotId 不一致 | Recovery 拒绝 |
| WR-006 | 同一 TransferId 重试 | 返回相同 Commit 结果 |
| WR-007 | 同一设备存在活动漂移记录 | 唯一约束阻止重复 |
| WR-008 | 人工修复无 OperationId | 返回 AUDIT-422 |
| WR-009 | Reconciliation 后 StateHash 校验 | Hash 一致才允许 Resume |
| WR-010 | Replay 使用当前最新布局 | 必须失败 |
| WR-011 | 72 小时连续运行 | 无孤儿漂移记录、锁和事件缺口 |
| WR-012 | SignalR/Query 追踪 | 可由 CorrelationId 反查完整链路 |

## P0 门禁

WR-001～WR-006、WR-009、WR-010 任一失败，不得进入 PLC/HIL 联调。

## 证据要求

每次失败必须保留：RunId、SimTick、DeviceId、CommandId、TransferId、LayoutSnapshotId、BindingSnapshotId、OccupancyVersion、ProcessImageVersion、EventSequence、前后 StateHash。
