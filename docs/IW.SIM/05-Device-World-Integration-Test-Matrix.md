# Part 05：Device Runtime—World Model 集成测试矩阵

## 1. 测试范围与固定输入

覆盖 `CommandGateway → Admission → ResourceLock → DeviceRuntime → OccupancyTransfer → EventStore → Snapshot/Replay`。每个用例固定 `ProjectVersion、RunId、Seed、InputSet`，并记录 `CommandId、TransferId、DeviceId、CargoId、SimTick、CorrelationId`。

## 2. 测试矩阵

| 编号 | 场景 | 前置条件 | 关键动作 | 预期结果 |
|---|---|---|---|---|
| DW-001 | 正常输送完成 | Cargo 在 FromPlace，ToPlace 可用 | 提交 Move | Command Succeeded，OccupancyVersion +1 |
| DW-002 | 重复命令 | 相同 CommandId/RequestHash | 重试提交 | 返回原结果，不重复运动 |
| DW-003 | RequestHash 冲突 | 相同 CommandId/不同 Hash | 重试提交 | `DEV-CMD-422`，状态不变 |
| DW-004 | 设备版本冲突 | ExpectedVersion 过期 | 提交命令 | 拒绝，DeviceVersion 不变 |
| DW-005 | 目标位置竞争 | 两命令竞争同一 ToPlace | 同 Tick 并发 Claim | 仅一个 Commit 成功 |
| DW-006 | 货载竞争 | 同一 Cargo 两活动命令 | 并发执行 | 第二命令 `WORLD-OCC-409-CARGO` |
| DW-007 | 锁租约过期 | LeaseUntilTick 已过 | 继续执行 | LockExpired，命令进入恢复/失败 |
| DW-008 | Commit 中断 | Prepared 后进程停止 | 恢复运行 | 只产生一个 Commit 或补偿结果 |
| DW-009 | 事件持久化失败 | Occupancy 已提交 | Outbox 补发 | 不回滚位置事实，不重复动作 |
| DW-010 | 事件重投 | 相同 EventId 重复消费 | 消费事件 | Device/Occupancy 版本不重复递增 |
| DW-011 | Snapshot 恢复 | 存在 Committed Snapshot | 恢复活动命令和锁 | 状态与 Snapshot 一致 |
| DW-012 | 回放一致性 | 历史 InputSet | Replay | Event/State/Result Hash 一致 |
| DW-013 | 取消竞态 | 命令处于 Executing 边界 | Cancel 与执行并发 | 结果确定且不可二次提交 |
| DW-014 | 多设备接驳 | Conveyor→Lift→Turntable | 连续 Transfer | 串行可追溯，无双重占用 |
| DW-015 | 传感器误触发 | 输入与位置事实冲突 | 注入输入 | 只记诊断，不直接改 Occupancy |
| DW-016 | 并行设备执行 | 不同 DeviceId | 同 Tick 并行 | 独立锁域，排序稳定 |

## 3. 性能与稳定性门槛

- 100 个并发设备、30ms 周期下命令准入 P95 < 50ms。
- 同一 Seed/InputSet 重复运行 100 次，Transfer 结果和 Hash 100% 一致。
- 连续 72 小时无未解释的活动锁泄漏。
- 事件重放吞吐不低于实时产生速率 2 倍。
- 任何阻断条件出现时，Run 自动进入 Degraded/Failed，不得静默继续。

## 4. 失败分类与发布门禁

- 配置类失败：禁止进入 Running。
- 业务冲突：只拒绝当前命令，不污染 World State。
- 基础设施失败：保留已提交事实，进入 Degraded 并补发 Outbox。
- Hash 不一致：禁止自动继续，必须生成阻断性诊断。
- Transfer 失败却产生 `CargoTransferCompleted`、重复改变 OccupancyVersion、Snapshot 恢复丢失活动锁、事件重投造成二次设备动作，任一发生即阻断发布。

## 5. 工程实现映射

- DW-001～006：命令准入、幂等、版本和竞争控制。
- DW-007～010：锁租约、Outbox、事件幂等和故障注入。
- DW-011～013：Snapshot/Replay/取消恢复。
- DW-014～016：多设备接驳、观测隔离和并发性能。
