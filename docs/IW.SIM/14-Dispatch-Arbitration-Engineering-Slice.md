# Slice K：Dispatch Arbitration 工程实施切片

## 目标

把 Task/MCS/WCS 的调度请求收敛到唯一仲裁入口，并打通 Route、Traffic Reservation、DeviceCommand 和 Occupancy Commit。

## 交付任务

1. 创建 `Logistics.Simulation.Dispatch.Contracts`。
2. 实现 `IDispatchArbiter`、`IDeadlockDetector`。
3. 创建 `dispatch_arbitration` EF Core Migration。
4. 实现稳定排序、AgingScore 和 Deadline 规则。
5. 实现资源等待图和环检测。
6. 将 Reservation 创建改为只接受 ArbitrationId。
7. 为抢占、重排、死锁和补发增加事件/审计。
8. 接入 DA-001～DA-015 测试矩阵。

## 完成定义

- API、WCS、PLC、HIL 均不得旁路仲裁。
- 同一 Cargo 无并发执行资格。
- 任意资源环可检测、可解释、可重放。
- Reservation 与 Arbitration 一致。
- `TaskCompleted` 仍必须等待 Occupancy Commit。
- P0 测试全部通过后才允许 WCS/HIL 联调。
