# Part 02：World Snapshot 与 Occupancy Authority

## 1. 目标

统一 LayoutSnapshot、DeviceBindingSnapshot、WorldStateSnapshot 与 Occupancy 的版本、读取和提交边界，避免 Runtime、Route、Transfer、Recovery、Replay 各自持有空间/位置副本。

## 2. 权威层级

```text
LayoutSnapshot
  └─ DeviceBindingSnapshot
       └─ WorldStateSnapshot
            └─ OccupancyIndex
```

- `LayoutSnapshot`：空间拓扑权威。
- `DeviceBindingSnapshot`：设备、能力、点位与空间绑定权威。
- `WorldStateSnapshot`：Run 内设备运行态、货载、锁和 Occupancy 的恢复点。
- `OccupancyIndex`：当前 Place/Cargo 互斥关系的唯一运行时视图。

Telemetry、SignalR 和渲染缓存不得作为权威来源。

## 3. C# 契约

```csharp
public interface IWorldSnapshotAuthority
{
    ValueTask<WorldSnapshotDescriptor> GetAsync(
        Guid runId,
        CancellationToken cancellationToken);

    ValueTask<WorldSnapshotValidationResult> ValidateAsync(
        WorldSnapshotReference reference,
        CancellationToken cancellationToken);
}

public interface IOccupancyAuthority
{
    ValueTask<OccupancyReadResult> ReadAsync(
        Guid runId,
        Guid placeId,
        CancellationToken cancellationToken);

    ValueTask<OccupancyCommitResult> CommitAsync(
        OccupancyCommitRequest request,
        CancellationToken cancellationToken);
}
```

## 4. 版本不变量

1. `WorldStateSnapshot.LayoutSnapshotId` 必须与 Run 初始化事实一致。
2. `WorldStateSnapshot.DeviceBindingSnapshotId` 必须与所有 DeviceRuntime 实例一致。
3. `OccupancyVersion` 单调递增，不得回退。
4. `StateHash` 必须覆盖设备、货载、锁、Occupancy 和版本游标。
5. 任何 Transfer Commit 必须携带 `LayoutSnapshotId`、`BindingSnapshotId`、`ExpectedOccupancyVersion`。

## 5. Occupancy Commit 时序

```text
Read expected version
  → validate From/To/Cargo ownership
  → claim locks
  → write occupancy delta
  → increment OccupancyVersion
  → append CargoTransferCommitted
  → enqueue Outbox
  → release/renew locks
```

Commit 与事件事实必须在同一业务事务内完成；Outbox 发布失败不得重新执行 Commit。

## 6. 错误码

| Code | 含义 | 处理 |
|---|---|---|
| WORLD-SNAPSHOT-409 | Snapshot 版本冲突 | 拒绝恢复/绑定 |
| WORLD-OCC-409 | OccupancyVersion 冲突 | 重新读取并重算 |
| WORLD-OCC-423 | 位置或货载被占用 | 进入等待/拒绝 |
| WORLD-HASH-422 | StateHash 不一致 | 终止 Replay |

## 7. 工程验收

- 恢复使用当前最新 Layout 时必须失败。
- 两次相同输入 Replay 得到相同 `StateHash`。
- Transfer 未 Commit 时不得出现目标 Occupancy。
- 同一 Cargo 同时只能有一个 ActiveTransfer。
- Snapshot 恢复后 `OccupancyVersion` 不得回退。
