# Part02 World Model：Occupancy 一致性契约

## 1. 唯一事实

`WorldOccupancy` 是货载当前位置、占用关系和可搬运状态的唯一事实来源。Device Runtime 不得维护第二套可写位置真相；设备内部可保留计算缓存，但必须通过 Occupancy 事务提交。

## 2. 数据模型

```csharp
public sealed record OccupancyClaim(
    Guid RunId,
    Guid CargoId,
    Guid? FromPlaceId,
    Guid ToPlaceId,
    Guid CommandId,
    long ExpectedOccupancyVersion,
    long SimTick);

public sealed record OccupancyCommit(
    Guid CargoId,
    Guid? FromPlaceId,
    Guid ToPlaceId,
    long OccupancyVersion,
    long SimTick,
    string StateHash);

public interface IWorldOccupancyService
{
    ValueTask<OccupancyCommit> CommitAsync(OccupancyClaim claim, CancellationToken ct);
    ValueTask<OccupancySnapshot> ReadAsync(Guid runId, Guid cargoId, CancellationToken ct);
}
```

## 3. 提交规则

1. FromPlace 必须仍然持有 CargoId；否则 `WORLD-OCC-409`。
2. ToPlace 必须可占用且未被其他 Cargo 占用；否则 `WORLD-OCC-423`。
3. 提交按 `SimTick -> CommandId` 确定性排序。
4. 成功提交同时产生 `CargoOccupancyChanged` 事件。
5. Event、Snapshot、Result 使用同一个 OccupancyVersion 和 StateHash。

## 4. 异常恢复

占用提交成功但事件写入失败时进入 `WorldDegraded`，由 Outbox 补发；禁止回滚已提交 Occupancy。Snapshot 恢复必须校验 StateHash，不一致则停止自动恢复并要求人工诊断。

## 5. 验收

- 两个命令竞争同一 ToPlace 时只有一个成功。
- Replay 后所有 Cargo 的 OccupancyVersion 与原运行一致。
- 传感器反馈不得直接写 Occupancy，必须通过 Device Runtime 的动作完成事件提交。
- 货载跨多设备接驳时只产生一个最终 Occupancy 提交链。
