# ADR-023：World Fact Authority 与 Reconciliation 边界

## 状态

Accepted

## 背景

Device Runtime、PLC Feedback、Telemetry、Route、Occupancy 和 Recovery 都会产生或读取状态。如果允许观测结果直接覆盖 World Fact，将导致位置、货载、锁和回放结果不可确定。

## 决策

1. LayoutSnapshot 是空间拓扑权威。
2. DeviceBindingSnapshot 是设备—能力—点位—空间绑定权威。
3. OccupancyAuthority 是货载位置事实权威。
4. Telemetry/ProcessImage 只能作为观测输入，不能直接覆盖 World Fact。
5. 所有不一致进入 `world_reconciliation_record`，由 Reconciliation Policy 决定等待、降级、恢复或失败。
6. 完成类事件必须以 Occupancy Commit 为前置事实。
7. Recovery/Replay 必须使用原始 LayoutSnapshotId 和 BindingSnapshotId。

## 后果

- 增加对账记录和状态管理成本。
- 需要统一版本、Hash、CorrelationId 和 OperationId。
- 设备、PLC、World、Data、Recovery 的边界更清晰，可测试和可回放。

## 拒绝的替代方案

- 用最新 Telemetry 覆盖 Occupancy。
- 用 PLC 到位信号直接生成 CargoAtDestination。
- Recovery 时读取当前最新 Layout/Device 配置。
