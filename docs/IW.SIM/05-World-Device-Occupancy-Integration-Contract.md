# Part 05：World、Device Runtime 与 Occupancy 集成契约

## 1. 边界

Device Runtime 负责设备状态演化和动作执行；World Model 负责空间与货载事实；Occupancy Authority 负责位置互斥提交。Device Runtime 不直接修改 Occupancy 表或 Process Image。

## 2. 接口

```csharp
public interface IDeviceWorldCoordinator
{
    ValueTask<DeviceWorldExecutionResult> ExecuteAsync(
        DeviceExecutionRequest request,
        CancellationToken cancellationToken);
}

public interface ITransferCompletionPolicy
{
    CompletionDecision Evaluate(
        DeviceExecutionState execution,
        OccupancyCommitResult occupancy,
        ProcessImageFrame feedback);
}
```

## 3. 完成条件

设备动作只有同时满足以下条件才允许发布 Completed：

1. DeviceExecutionState = `TransferPending` 或 `Executing` 的合法终态。
2. `OccupancyCommitResult = Committed`。
3. `OccupancyVersion` 比执行开始时递增。
4. ProcessImageFrame 的 `ProcessImageVersion` 不落后。
5. LayoutHash、BindingHash 与 Run 初始化事实一致。

## 4. 失败路径

```text
Device Execute
  → TransferPending
  → Occupancy Commit
      ├─ Committed → Feedback → Completed
      ├─ Conflict → Recovering
      ├─ Timeout → Degraded
      └─ InvalidSnapshot → Failed
```

## 5. 禁止事项

- 禁止 DeviceRuntime 直接写 `CargoAtDestination`。
- 禁止 PLC Mapper 绕过 Command Admission 创建设备动作。
- 禁止 Occupancy Commit 成功前释放目标位置锁。
- 禁止在 Recovery/Replay 中使用当前最新 DeviceBindingSnapshot。

## 6. 事件契约

`DeviceExecutionChanged` 必须携带：`RunId、DeviceId、CommandId、TransferId、SimTick、DeviceVersion、BindingSnapshotId、SchemaVersion`。

`CargoTransferCommitted` 必须携带：`TransferId、CargoId、FromPlaceId、ToPlaceId、OccupancyVersion、LayoutSnapshotId、StateHash`。

## 7. 验收

- 设备动作成功但 Occupancy 冲突时不得产生 Completed。
- 同一 TransferId 重试必须返回同一提交结果。
- Feedback 版本落后时不得覆盖已提交过程映像。
