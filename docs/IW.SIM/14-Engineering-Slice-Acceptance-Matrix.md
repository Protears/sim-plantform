# Part 14：工程切片验收矩阵

## 1. Slice A：纯仿真

| 场景 | 预置 | 关键动作 | 断言 |
|---|---|---|---|
| A-01 输送机转移 | 一件 Cargo、空目标 Place | 执行 Move | OccupancyVersion +1，事件顺序连续 |
| A-02 目标竞争 | 两命令同一 ToPlace | 并发提交 | 仅一条 Transfer Commit 成功 |
| A-03 重放 | 已封存 Run | Replay | Device/Occupancy/Result Hash 一致 |

## 2. Slice B：PLC 联调

| 场景 | 关键断言 |
|---|---|
| B-01 RisingEdge | 一个边沿只生成一个 CommandId |
| B-02 Level/Cooldown | 冷却期内不重复下发 |
| B-03 反馈延迟 | Device 状态只在下一 InputCommit 可见 |
| B-04 Scan Overrun | 超时周期不提交部分输出 |

## 3. Slice C：HIL

| 场景 | 关键断言 |
|---|---|
| C-01 旧 Epoch | 旧帧被拒收且不改 Process Image |
| C-02 Ack 超时 | Critical Output 进入 FailSafe |
| C-03 断线恢复 | 重连、对账完成前禁止普通输出 |
| C-04 半包粘包 | 重组帧数量与业务序列一致 |

## 4. 质量门槛

- 单元测试：Domain/Mapper/StateMachine 覆盖率 ≥ 90%。
- 集成测试：命令、Occupancy、Outbox、Replay 全部通过。
- 性能：100 个 PLC/设备并发时 Tick 漂移 ≤ 60ms。
- 稳定性：72 小时无重复 CommandId、孤儿锁、Event Sequence 断裂。
- 发布：架构测试、迁移测试、契约测试、回放测试全部绿灯。

## 5. 失败处理

任一 Critical 场景失败，禁止进入联调分支；保留 RunId、CommandId、TransferId、Sequence、CorrelationId、SnapshotId 作为缺陷证据。
