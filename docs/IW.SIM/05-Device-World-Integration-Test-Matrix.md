# Part05/Part02 设备运行时与世界模型集成测试矩阵

## 1. 适用范围

覆盖 CommandGateway、ResourceLock、Device Runtime、WorldOccupancy、Event Store、Snapshot/Replay 的集成边界。所有测试使用固定 ProjectVersion、Seed 和 InputSet。

## 2. 测试矩阵

| 编号 | 场景 | 前置条件 | 预期结果 | 关键断言 |
|---|---|---|---|---|
| DW-001 | 正常输送完成 | Cargo 位于 FromPlace | 命令成功并提交 Occupancy | OccupancyVersion +1 |
| DW-002 | 重复 CommandId | 首次已完成 | 返回原结果 | 不产生第二次运动 |
| DW-003 | 设备版本冲突 | ExpectedVersion 过期 | 拒绝命令 | 状态无变化 |
| DW-004 | 目标库位竞争 | 两命令竞争同一 Place | 仅一个成功 | 另一条为 OCC-423 |
| DW-005 | 资源锁超时 | LeaseUntilTick 已过 | LockExpired + Degraded | 无孤儿锁 |
| DW-006 | 事件持久化失败 | Occupancy 已提交 | Outbox 待补发 | 不回滚位置事实 |
| DW-007 | Snapshot 恢复 | 存在 Committed Snapshot | 继续运行 | Hash 校验通过 |
| DW-008 | 回放一致性 | 历史 InputSet | 重演完成 | Event/State/Result Hash 一致 |
| DW-009 | 取消竞态 | 命令处于 Executing 边界 | 确定性成功/失败 | 同一输入重复结果相同 |
| DW-010 | 多设备接驳 | Conveyor→Lift→Turntable | 链路完成 | 不出现双重占用 |
| DW-011 | 传感器误触发 | 输入与位置事实冲突 | 记录诊断 | 不直接改 Occupancy |
| DW-012 | 并行设备执行 | 不同 DeviceId | 可并行 | 同 Tick 排序稳定 |

## 3. 非功能门槛

- 100 个并发设备下命令接受 P95 < 50ms（不含数据库冷启动）。
- 单 Run 事件顺序 100% 可重放。
- 72 小时运行无内存持续增长。
- 锁、Occupancy、Event Store 三者的关键 ID 可链路追踪。

## 4. 失败分类

- 配置类失败：测试不得进入 Running。
- 业务冲突：只拒绝当前命令，不污染 World State。
- 基础设施失败：进入 Degraded，保留事实并触发 Outbox/恢复。
- Hash 不一致：禁止自动继续，必须生成阻断性诊断。
