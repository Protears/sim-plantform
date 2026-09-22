# Part 14：World Authority 工程实施切片

## Slice I：World Fact Authority 与 Reconciliation

### 目标

把 LayoutSnapshot、DeviceBindingSnapshot、OccupancyAuthority、WorldReconciliation 和 Recovery/Replay 接入统一工程闭环。

### 项目切分

```text
Logistics.Simulation.World.Contracts
Logistics.Simulation.World
Logistics.Simulation.World.Data
Logistics.Simulation.World.Tests
Logistics.Simulation.World.IntegrationTests
```

### 任务

1. 实现 `IWorldSnapshotAuthority`。
2. 实现 `IOccupancyAuthority` 和版本校验。
3. 建立 `world_reconciliation_record` Migration。
4. 实现 `IWorldReconciliationService` 状态机。
5. 在 `IDeviceWorldCoordinator` 中接入 Transfer Commit 前置条件。
6. 将 LayoutSnapshotId、BindingSnapshotId、OccupancyVersion、StateHash 写入事件和 Snapshot。
7. 增加 WR-001～WR-012 自动化测试。
8. 将 P0 对账门禁接入 Operational Readiness。

### 完成定义

- Device Runtime 不再直接写 Occupancy。
- 位置冲突可进入 Degraded/Failed，并生成追加审计。
- Recovery/Replay 使用原始空间与设备绑定快照。
- 两次相同输入得到相同 StateHash。
- 失败证据可沿 CorrelationId 反查至 EventSequence。

### 依赖规则

- `World` 依赖 Domain、Contracts，不依赖 API、PLC SDK。
- `World.Data` 是唯一 EF Core 实现位置。
- `DeviceRuntime` 只能调用 `IOccupancyAuthority`，不能访问 `World.Data`。
- `Reconciliation` 不得绕过 Kernel 直接推进 SimTick。
