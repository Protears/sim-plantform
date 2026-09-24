# Dispatch Arbitration Test Matrix

| ID | 场景 | 预期 |
|---|---|---|
| DA-001 | 相同 SubTask 重复提交 | 返回同一 ArbitrationId |
| DA-002 | 同 Cargo 并发获得资格 | 第二个被拒绝 |
| DA-003 | 依赖未完成 | 不得进入 Granted |
| DA-004 | WorldVersion 过期 | 返回 409 |
| DA-005 | 资源按逆序申请 | 架构测试失败 |
| DA-006 | 形成等待环 | 生成 DeadlockDetected |
| DA-007 | 选择受害者 | 只能选择可抢占任务 |
| DA-008 | Executing 抢占 | 返回 423 |
| DA-009 | Deadline 已过 | 返回 429 |
| DA-010 | Reservation 创建失败 | Arbitration 保持 WaitingResource |
| DA-011 | Granted 后 Outbox 重试 | 不得重复创建 Reservation |
| DA-012 | SignalR 断线补发 | Sequence 连续且客户端幂等 |
| DA-013 | Replay | Arbitration/Deadlock 结果一致 |
| DA-014 | 72 小时稳定性 | 无孤儿 Arbitration、锁和队列记录 |
| DA-015 | HIL 联调 | 任何 P0 失败阻断发布 |

## 通过门禁

DA-001～DA-008、DA-011、DA-013 为 P0；任一失败不得进入 WCS/HIL 联调。
