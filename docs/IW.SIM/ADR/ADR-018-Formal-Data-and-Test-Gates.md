# ADR-018：正式运行态数据模型与测试门禁

- 状态：Accepted
- 日期：2026-09-17
- 关联章节：Part03、Part05、Part06、Part10、Part11、Part14

## 背景

已有文档分别定义了 Kernel、设备命令、Occupancy、Outbox、PLC 反馈和工程门禁，但如果没有统一 Schema、迁移规则和强制测试，运行时实现容易产生重复事实、孤儿锁、事件补发缺口和 Replay 分叉。

## 决策

1. Command、Transfer、Lock、Event、Outbox、Snapshot 使用 Part10 正式 Schema。
2. 业务事实提交采用“事务提交后发布”规则；Dispatcher 不执行领域动作。
3. Run Event 查询和 SignalR 补发以 Event Sequence 为唯一游标。
4. Architecture、Contract、Migration、Replay、Stability 五类测试作为发布强制门禁。
5. SchemaVersion 不兼容时，运行时不得进入 Running。

## 影响

- 增加正式数据迁移和回滚责任。
- Application、Device Runtime、World Model 必须通过 Store/Port 访问数据。
- CI 时间增加，但可以在进入 HIL 联调前发现跨模块不一致。
- 历史文档中直接使用临时表或 Telemetry 恢复事实的做法视为废弃。

## 拒绝的替代方案

- 仅使用内存状态，不满足恢复和 Replay。
- 由 SignalR 消息作为事实来源，不保证断线和重放。
- 由业务模块各自分配 Event Sequence，无法保证全局顺序。
