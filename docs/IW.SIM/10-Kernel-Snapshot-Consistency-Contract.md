# Part 10：Kernel Snapshot 一致性契约

## 1. 目标

定义 Snapshot 在 Kernel Tick 边界的冻结、写入、校验和恢复规则，确保 Device Runtime、PLC Process Image、Occupancy、Scheduler Queue、Input Cursor 可被整体恢复，避免“事件已写但运行态未写”或“运行态已恢复但输入游标缺失”。

## 2. 一致性点

Snapshot 只能在以下边界创建：

```text
InputCommit 完成
→ PlcScan 完成
→ DeviceExecute 完成
→ OccupancyCommit 完成
→ EventAppend 完成
→ Snapshot Freeze
```

不得在阶段中间创建可提交 Snapshot。

## 3. Snapshot 内容

| 分区 | 必填字段 |
|---|---|
| Run | RunId、SimTick、RunState |
| Kernel | NextSequence、SchedulerQueue、Phase |
| Input | InputCursor、ClockEpoch、LastExternalSequence |
| PLC | ProcessImageVersion、InputHash、OutputHash |
| Device | DeviceVersion、ActiveCommandId、LockLease |
| World | OccupancyVersion、StateHash |
| Integrity | BaseEventSequence、SnapshotHash、SchemaVersion |

## 4. C# 接口

```csharp
public interface ISimulationSnapshotCoordinator
{
    ValueTask<SnapshotWriteResult> CaptureAsync(SnapshotCaptureContext context, CancellationToken ct);
    ValueTask<SnapshotRestoreResult> RestoreAsync(Guid runId, Guid snapshotId, CancellationToken ct);
}

public sealed record SnapshotCaptureContext(
    Guid RunId,
    long SimTick,
    long BaseEventSequence,
    string ProcessImageHash,
    string WorldStateHash,
    string InputCursor,
    ReadOnlyMemory<byte> SchedulerState);
```

## 5. 写入状态机

```text
Requested → Freezing → Persisting → Verifying → Committed
                                      └──────→ Failed
```

- `Freezing`：阻止当前 Run 的下一次 Tick 提交，不阻塞异步 I/O 接收。
- `Persisting`：写入 Snapshot 主体和索引。
- `Verifying`：重新计算 Hash，并验证 `BaseEventSequence` 存在。
- `Committed`：对外可见。

## 6. 恢复顺序

1. 校验 SnapshotHash 和 SchemaVersion。
2. 恢复 Event Cursor、Input Cursor、ClockEpoch。
3. 恢复 World Occupancy 和 Device Runtime。
4. 恢复 PLC Process Image 和 Scheduler Queue。
5. 重建内存索引与锁租约。
6. 以 `BaseEventSequence + 1` 开始 Replay。
7. 验证首个恢复 Tick 的 Hash。

## 7. 验收项

- Snapshot 不得引用未提交的 Event Sequence。
- 恢复后 `OccupancyVersion`、`ProcessImageVersion` 单调不回退。
- 恢复后同一输入重放产生相同 Result Hash。
- Snapshot 写入失败不得改变 Run 的业务状态。
- 72 小时运行中 Snapshot 可按策略恢复并通过一致性校验。