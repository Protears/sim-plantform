# ADR-012：Occupancy 作为货载位置唯一权威

- 状态：Accepted
- 适用：Part02 World Model、Part05 Device Runtime、Part10 Data

## 背景

历史设计允许设备运行时、货载追踪器和 World Model 同时保存位置，导致多设备接驳、恢复和回放时出现位置分叉。

## 决策

1. `WorldOccupancy` 是唯一可写位置权威。
2. Device Runtime 只提交 OccupancyClaim，不直接修改位置表。
3. 传感器只产生输入和观测，不产生最终位置事实。
4. OccupancyCommit 与业务事件共享同一事务边界和版本号。
5. Replay 以 Occupancy 事件链重建位置，禁止从 Telemetry 推断。

## 后果

优点：可审计、可恢复、可重放、跨设备接驳一致。代价：设备完成动作必须经过一次提交协议，不能通过简单内存赋值完成。

## 约束

- 禁止新增第二套位置真相表。
- 任何新设备类型必须实现 `IWorldOccupancyService` 接入测试。
- 发现 StateHash 不一致时禁止继续自动运行。
