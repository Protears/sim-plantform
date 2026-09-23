# Task/MCS/WCS 编排验收矩阵

| ID | 场景 | 预期 | 门禁 |
|---|---|---|---|
| TO-001 | 重复 TaskId + 相同 RequestHash | 返回原 Task/PlanHash | P0 |
| TO-002 | 相同 TaskId + 不同 RequestHash | `TASK-422` | P0 |
| TO-003 | 依赖图存在环 | 拒绝规划 | P0 |
| TO-004 | 依赖任务未完成 | 保持 Ready，不得 Dispatch | P0 |
| TO-005 | WorldVersion 过期 | `409`，要求 Replan | P0 |
| TO-006 | Cargo 已有活动任务 | `TASK-423` | P0 |
| TO-007 | Route 无可达路径 | `WCS-422` | P0 |
| TO-008 | Reservation 竞争 | 不生成 DeviceCommand | P0 |
| TO-009 | Command 完成但 Transfer 未 Commit | 不得 TaskCompleted | P0 |
| TO-010 | Transfer Commit 成功 | 写 TaskCompleted + Outbox | P0 |
| TO-011 | Cancel 在 Executing 前 | 释放 Reservation | P1 |
| TO-012 | Cancel 在 TransferPending | 进入补偿，不得伪造完成 | P0 |
| TO-013 | SignalR 断线 | 按 EventSequence 补发 | P1 |
| TO-014 | Replay | 使用原始 TaskSnapshot/RouteHash | P0 |
| TO-015 | 72h 稳定运行 | 无孤儿任务、Reservation、锁 | P0 |

## 通过标准

- P0 全部通过才能进入 WCS/HIL 联调。
- TO-009、TO-010、TO-012 必须提供 EventSequence、TransferId、OccupancyVersion 证据。
- 任一任务状态与 Occupancy 状态不一致，运行标记为 Degraded，禁止自动发布完成类事件。
