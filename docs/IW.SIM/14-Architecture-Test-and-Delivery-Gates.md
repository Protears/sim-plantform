# Part 14：架构测试与交付门禁

## 1. 目标

把文档中的模块边界、时间权威、幂等、事件、数据责任和恢复约束转化为自动化门禁，防止实现阶段重新引入跨层依赖和事实分叉。

## 2. 架构测试

使用 NetArchTest 或自定义 Roslyn Analyzer 验证：

- Domain 不引用 `Microsoft.EntityFrameworkCore`、`ASP.NET Core`、PLC SDK。
- SimulationKernel 不引用 Data、Api、SignalR。
- DeviceRuntime 不引用 Controller、DbContext。
- Communication/Hil 不引用 WorldModel 写模型。
- Api 不直接引用具体设备实现，只引用 Application Port。

## 3. 契约测试矩阵

| 编号 | 场景 | 期望 |
|---|---|---|
| AT-001 | 相同 CommandId 重试 | 返回原结果，不新增命令 |
| AT-002 | RequestHash 冲突 | 422，原命令不变 |
| AT-003 | Sealed Run 写命令 | 409/423，事实表无新增 |
| AT-004 | 同目标位并发 Transfer | 仅一个 Commit |
| AT-005 | Outbox 重投 | 消费幂等，无重复状态变化 |
| AT-006 | Snapshot 恢复 | 活动锁/Transfer/Command 可重建 |
| AT-007 | Replay | Event/State/Result Hash 一致 |
| AT-008 | PLC RisingEdge | 一个边沿一个 Intent |
| AT-009 | 旧 Epoch 输入 | 丢弃并生成诊断事件 |
| AT-010 | Critical Ack 超时 | 触发 FailSafe |

## 4. 性能与稳定性门禁

- 100 个并发设备、30ms Tick 下 Kernel 漂移不超过 60ms。
- Command Admission P95 小于 20ms（不含异步 Outbox 发布）。
- Occupancy Commit P95 小于 10ms，冲突率可观测。
- 连续 72 小时运行无未释放锁、无重复 CommandId、无事件序列断裂。
- SignalR 断线重连后按 LastSequence 补齐，不允许丢失关键事件。

## 5. 发布门禁

未通过架构测试、契约测试、迁移测试和 Replay 测试的版本不得进入联调分支。设计变更必须同步更新对应 ADR、DTO/Event Contract 和测试矩阵；只改实现不改契约视为不完整变更。

## 6. 直接工程任务

1. 建立 `ArchitectureTests` 项目。
2. 建立 `ContractTests` 的固定测试基类。
3. 在 CI 中执行空库迁移、快照恢复、Replay Hash、并发 Transfer 测试。
4. 将门禁结果写入发布制品 Manifest。
