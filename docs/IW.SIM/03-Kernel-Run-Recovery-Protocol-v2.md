# Part 03：Kernel Run Recovery Protocol v2

## 1. 目标

定义运行中断、Snapshot 校验、事实恢复、队列恢复、Replay 和重新进入 Running 的统一协议。Recovery 不允许推测性修复；无法证明事实完整时必须转 Failed。

## 2. 状态机

```text
Created → Ready → Running → Pausing → Paused
Paused → Recovering → Replaying → Ready/Running
Recovering → Degraded → Recovering
Recovering → Failed
Running → Sealed
```

## 3. C# 接口

```csharp
public interface IKernelRecoveryCoordinator
{
    Task<RecoveryResult> RecoverAsync(
        RecoveryRequest request,
        CancellationToken cancellationToken);
}

public sealed record RecoveryRequest(
    Guid RunId,
    Guid SnapshotId,
    long ExpectedBaseEventSequence,
    string ExpectedSnapshotHash,
    bool AllowFullReplay);

public sealed record RecoveryResult(
    bool Succeeded,
    Guid RunId,
    long RestoredSimTick,
    long RestoredEventSequence,
    string StateHash,
    IReadOnlyList<string> Diagnostics);
```

## 4. 恢复顺序

1. 校验 Snapshot 状态为 Committed。
2. 校验 SnapshotHash、BaseEventSequence、SchemaVersion。
3. 恢复已提交 Event/Occupancy/Command/Lock 事实。
4. 恢复 Input Cursor、ClockEpoch、RandomState。
5. 重建 World/Device/PLC Process Image 内存索引。
6. 恢复 Scheduler Queue，按原排序键重排。
7. 对未完成输入执行 Replay；不得调用外部网络。
8. 校验 DeviceHash、OccupancyHash、ResultHash。
9. 成功后进入 Ready，再由 Application 显式 Resume。

## 5. 关键不变量

- `RestoredSimTick >= Snapshot.SimTick`，不得回退。
- `EventSequence` 不得倒退或重复。
- 旧 ClockEpoch 输入全部拒收。
- Recovery/Replaying 期间不得发布 Completed、CargoTransferCommitted 等对外完成事件。
- 原始 Run 只能恢复和继续，不允许在 Replay 模式下写入原始事实。

## 6. 错误码

| 错误码 | 条件 | 动作 |
|---|---|---|
| KERNEL-RECOVERY-001 | Snapshot 非 Committed | Failed |
| KERNEL-RECOVERY-002 | Hash 不匹配 | Failed |
| KERNEL-RECOVERY-003 | EventSequence 缺口 | FullReplay 或 Failed |
| KERNEL-RECOVERY-004 | SchemaVersion 不兼容 | 拒绝启动 |
| KERNEL-RECOVERY-005 | Epoch 过期 | 丢弃输入并审计 |
| KERNEL-RECOVERY-006 | 恢复后版本回退 | Failed |

## 7. 验收

- 中断发生在 Occupancy Commit 前后时，恢复结果只能落在已提交事实边界。
- 同一 Snapshot 重复恢复结果 Hash 必须一致。
- 输入到达顺序变化但逻辑顺序相同，Replay Hash 必须一致。
- Recovery 失败时不得产生误导性的设备完成信号。