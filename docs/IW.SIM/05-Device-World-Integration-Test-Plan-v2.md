# Part05 设备运行时与 World Model 集成测试计划 v2

## 1. 目的

将命令准入、能力校验、资源锁、设备执行、货载转移、Occupancy 提交、事件幂等、快照恢复和回放统一为可执行测试入口。

## 2. 测试用例

| 编号 | 场景 | 预期 |
|---|---|---|
| DW2-001 | 首次命令 | Accepted→Completed，只有一个运动实例 |
| DW2-002 | 相同 CommandId 重试 | 返回原结果，不重复执行 |
| DW2-003 | RequestHash 冲突 | 拒绝并返回 DEV-CMD-422 |
| DW2-004 | ExpectedDeviceVersion 过期 | 拒绝，设备状态不变 |
| DW2-005 | 同一库位竞争 | 只有一个 Transfer Claim 成功 |
| DW2-006 | Occupancy Commit 冲突 | 转 TransferPending，不产生 Completed |
| DW2-007 | 锁租约到期 | 进入 Recovering，产生锁失效事件 |
| DW2-008 | Event 重投 | 幂等消费，不重复推进版本 |
| DW2-009 | Snapshot 恢复 | 恢复未完成 TransferContext，TransferId 不变 |
| DW2-010 | Deterministic Replay | Event/State Hash 一致 |
| DW2-011 | Conveyor→Lift→Turntable | 每次转移版本连续且可追踪 |
| DW2-012 | Sealed Run 新命令 | 拒绝，返回 RUN-STATE-409 |

## 3. 性能门槛

- 10,000 条命令回放顺序 100% 一致。
- 20,000 个通信设备配置下，锁和 Occupancy 不得串扰。
- 72 小时运行期间，未释放锁和重复事件不得持续增长。
- 关键设备执行路径禁止访问数据库。

## 4. 工程 DoD

新增设备类型必须提供 Capability、Command、Transfer、Event、Replay 五类自动化测试；缺少任一类不得进入集成分支。
