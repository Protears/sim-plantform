# ADR-025：Dispatch Arbitration 是唯一授权入口

- 状态：Accepted
- 日期：2026-09-24

## 背景

Task/MCS/WCS、Route、Traffic、DeviceCommand 和 Occupancy 之间若允许多个模块直接发放执行资格，会产生重复 Reservation、资源死锁、TaskCompleted 提前发布和 Replay 不一致。

## 决策

1. `IDispatchArbiter` 是进入 Traffic Reservation 的唯一授权入口。
2. WCS、PLC、HIL、API 不得直接创建 Reservation 或 DeviceCommand。
3. 抢占只针对未执行任务，且必须产生可重放事件和 Audit。
4. 所有资源按 `SafetyZone → Region → Track → Place → Device → PlcSession` 申请。
5. `Granted` 必须在 T-GRANT 事务提交后才对外发布。

## 后果

- 调度路径更集中，便于确定性 Replay 和审计。
- 需要额外维护 Arbitration 表和死锁检测。
- WCS 与 Device Runtime 之间必须通过 Application Port 交互。

## 约束

架构测试必须阻止 API、WCS、PLC Mapper、HIL 直接引用 `ITrafficReservationStore` 的写入实现。
