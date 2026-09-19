# Part 14：Operational Readiness 与发布门禁

## 1. 目标

把“文档可评审”推进为“运行可交付”：配置可校验、迁移可回滚、恢复可验证、审计可追踪、查询可补发、故障可升级。

## 2. 门禁矩阵

| Gate | 通过条件 | 阻断级别 |
|---|---|---|
| Config | SchemaVersion/MappingHash/Capability 全部通过 | Block |
| Architecture | 依赖白名单无违规 | Block |
| Migration | Expand/Backfill/Switch/Contract 脚本可回滚 | Block |
| Lifecycle | Run 状态机并发/幂等测试通过 | Block |
| Runtime | 30ms Tick、100 设备、Trace 完整 | Block |
| Recovery | Snapshot/Replay Hash 一致 | Block |
| Query | EventSequence 缺口检测和补发通过 | Block |
| Audit | 关键运维动作可追踪 | Block |
| Stability | 72 小时无孤儿锁、重复命令、序列断裂 | Block |

## 3. 运行前检查

1. Profile 已 Approved/Frozen。
2. SchemaVersion 与 Migration 兼容。
3. MappingHash、ProtocolProfile、Capability 清单已锁定。
4. EventStore、SnapshotStore、Outbox、AuditStore 可用。
5. Kernel、Ingress、Recovery Coordinator 健康。

## 4. 运行中指标

- `sim_tick_lag_ms`
- `input_queue_depth_ratio`
- `outbox_pending_count`
- `recovery_duration_ms`
- `trace_incomplete_count`
- `event_sequence_gap_count`
- `orphan_lock_count`
- `duplicate_command_count`
- `audit_write_failure_count`

## 5. 发布阻断规则

任何以下情况不得进入 HIL 联调或生产预演：

- Run Profile 未冻结；
- SchemaVersion 不兼容；
- Recovery 期间存在对外完成事件；
- Trace 缺少 CommandId/TransferId/OccupancyVersion 任一关键节点；
- EventSequence 存在未解释缺口；
- 审计写入失败且未进入 Degraded；
- 72 小时稳定性任一硬门槛失败。

## 6. 直接工程任务

- 将门禁矩阵转为 CI job 和运行前检查器。
- 建立故障注入：DB 超时、Outbox 堵塞、SignalR 断线、Snapshot 损坏、旧 Epoch 输入。
- 生成发布证据包：配置摘要、Schema 版本、测试报告、Recovery/Replay Hash、Trace 抽样和审计摘要。
