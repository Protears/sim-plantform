# Part02 World Model：货载接驳与 Occupancy Transfer 协议

## 1. 目标

定义 Device Runtime 将货载从一个 Place/TrackSegment 转移到另一个位置时，如何以声明、校验、提交三阶段修改世界状态。Occupancy 是唯一位置事实；传感器、设备动画和 Telemetry 都不能直接改写位置。

## 2. 三阶段协议

1. `Claim`：校验 From 仍由 Cargo 持有、To 可用、版本匹配。
2. `Prepare`：锁定 From/To/Cargo，形成不可变 TransferContext。
3. `Commit`：原子更新 Occupancy、CargoPosition、OccupancyVersion，并发布事件。

失败只能取消 Claim 或进入恢复，禁止部分提交。

## 3. C# 契约

```csharp
public sealed record TransferIntent(
    Guid TransferId,
    Guid RunId,
    Guid CargoId,
    Guid FromPlaceId,
    Guid ToPlaceId,
    long ExpectedOccupancyVersion,
    long TargetSimTick,
    string? CorrelationId);

public sealed record OccupancyCommitResult(
    Guid TransferId,
    bool Committed,
    long OccupancyVersion,
    string? ErrorCode,
    string StateHash);

public interface IWorldOccupancyTransfer
{
    OccupancyClaimResult Claim(TransferIntent intent, WorldOccupancySnapshot snapshot);
    OccupancyCommitResult Commit(TransferIntent intent, OccupancyClaim claim, SimTick tick);
    void Abort(Guid transferId, string reason, SimTick tick);
}
```

## 4. 不变量

- Cargo 不能同时占用两个互斥 Place。
- 一个 ToPlace 在同一 SimTick 只能成功一次 Claim。
- `ExpectedOccupancyVersion` 不匹配时不修改任何位置事实。
- Commit 成功后必须递增 OccupancyVersion，并产生 `CargoOccupancyChanged`。
- 多设备接驳必须通过同一 Transfer 协议，不得用设备私有字段传递位置。

## 5. 事件契约

```json
{
  "eventType": "CargoOccupancyChanged",
  "transferId": "uuid",
  "cargoId": "uuid",
  "fromPlaceId": "uuid",
  "toPlaceId": "uuid",
  "occupancyVersion": 42,
  "simTick": 1200,
  "stateHash": "sha256"
}
```

## 6. 并发与恢复

锁顺序固定为 `FromPlace -> ToPlace -> Cargo`。Snapshot 必须包含未完成 TransferContext、锁状态、OccupancyVersion 和 StateHash。恢复后只有在 TransferContext 校验通过时才能继续 Commit，否则转 Abort 并记录 `WORLD-OCC-409`。

## 7. 验收

- 两台设备竞争同一库位，仅一条 Transfer 成功。
- Commit 重放不得生成第二个 CargoOccupancyChanged。
- 设备动画完成但 Occupancy Commit 失败时，运行进入 Degraded，不得报告任务完成。
- Replay 后每个 SimTick 的 Occupancy Hash 与原运行一致。
