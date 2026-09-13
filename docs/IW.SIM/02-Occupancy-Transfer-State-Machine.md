# Part 02：Occupancy Transfer 状态机与一致性规则

## 1. 状态机

```text
IntentCreated → Claimed → Prepared → Committed
      │             │          │
      └─────────────┴──────────┴→ Rejected
                              │
                              └→ Compensating → Compensated
```

- `IntentCreated`：设备运行时表达货载转移意图。
- `Claimed`：目标位置已被本 Transfer 预占，版本校验成功。
- `Prepared`：源位置、目标位置、货载和设备锁均已确认。
- `Committed`：World Occupancy 原子更新完成。
- `Rejected`：版本、容量、占用或生命周期校验失败。
- `Compensating`：设备动作已发生但事实提交失败，进入补偿流程。
- `Compensated`：补偿完成，不得伪造为 Committed。

## 2. 一致性不变量

1. 一个 Run 内，一个 Cargo 只能有一个 Active Transfer。
2. 一个 Place 在同一 SimTick 内只能有一个成功 Claim。
3. Commit 必须校验 `ExpectedOccupancyVersion`。
4. Commit 成功后 `OccupancyVersion` 单调递增。
5. 传感器只提供观测，不直接写 Occupancy。
6. `CargoTransferCompleted` 只能由 Commit 成功路径产生。
7. Transfer 状态和 Occupancy 更新必须位于同一数据库事务或同一 Kernel 原子提交点。

## 3. 接口与错误

```csharp
public sealed record TransferIntent(
    Guid RunId, Guid TransferId, Guid CargoId,
    Guid? FromPlaceId, Guid ToPlaceId, Guid DeviceId,
    long ExpectedOccupancyVersion, long SimTick, Guid CorrelationId);

public sealed record TransferCommit(
    Guid RunId, Guid TransferId, long ExpectedOccupancyVersion,
    string StateHash, long SimTick);

public interface IWorldOccupancyTransfer
{
    ValueTask<TransferClaimResult> ClaimAsync(
        TransferIntent intent, CancellationToken cancellationToken);

    ValueTask<TransferCommitResult> CommitAsync(
        TransferCommit commit, CancellationToken cancellationToken);
}
```

错误码：`WORLD-OCC-409` 版本冲突；`WORLD-OCC-423` 目标位置不可占用；`WORLD-OCC-409-CARGO` 货载已有活动 Transfer；`WORLD-OCC-500` Commit 持久化失败。

## 4. 故障恢复

设备动作完成但 Commit 失败时，Runtime 必须保留 TransferContext 和设备事实，禁止直接重试 Commit 之外的设备动作。恢复顺序为：加载 Transfer → 校验锁 → 校验 OccupancyVersion → 幂等 Commit 或进入 Compensating。恢复期间禁止同一 Cargo 的新命令。

## 5. 测试门槛

- 两个设备竞争同一 ToPlace，只有一个进入 Committed。
- 重复 Commit 不产生第二个 OccupancyVersion。
- 进程中断发生在 Prepared 与 Commit 之间，恢复后结果唯一。
- 传感器抖动不得产生额外 Transfer。
