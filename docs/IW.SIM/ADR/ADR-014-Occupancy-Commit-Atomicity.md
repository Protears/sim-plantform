# ADR-014：Occupancy Commit 与设备完成事件的原子性

- 状态：Accepted
- 日期：2026-09-13
- 影响范围：Part02 World Model、Part05 Device Runtime、Part10 Data、Part11 Application

## 背景

设备动作完成并不等于货载位置事实已经完成。若先发布 `CargoTransferCompleted` 再提交 Occupancy，进程中断会造成事件与世界状态分叉；若只提交 Occupancy 而不可靠发布事件，下游投影和 WCS 会缺少事实通知。

## 决策

采用“事实先提交、事件可靠投递”的边界：

1. 在同一 Kernel 原子提交点或同一数据库事务中更新 `occupancy_transfer` 与 World Occupancy。
2. 只有 Commit 成功后才生成 `CargoTransferCommitted` 事件记录。
3. 事件与 Outbox 同一事务写入；发布失败只重试 Outbox，不重做设备动作。
4. 业务消费者以 `(RunId, EventId)` 幂等；位置事实以 `OccupancyVersion` 防止回退。
5. Commit 失败时 Transfer 进入 `Compensating` 或 `Rejected`，严禁发布 Completed。

## 结果

- 位置事实是唯一恢复依据。
- 事件可重复投递但不会重复改变事实。
- Snapshot/Replay 可以从 Transfer 和 Occupancy 的版本恢复。
- Part11 查询层必须把“命令完成”和“货载位置提交完成”区分为两个状态。

## 被拒绝方案

- 设备控制器直接发布 `CargoTransferCompleted`：会绕过 World Model 事实校验。
- 先发事件后异步写 Occupancy：会在故障时产生不可修复的事实分叉。
- 只依赖 Telemetry 推断位置：不可确定性回放且无法审计。

## 验收标准

- 任意 Commit 中断点恢复后，最多存在一个成功 Transfer。
- 重复 Outbox 投递不改变 OccupancyVersion。
- 查询接口能区分 `DeviceExecutionSucceeded` 与 `CargoTransferCommitted`。
