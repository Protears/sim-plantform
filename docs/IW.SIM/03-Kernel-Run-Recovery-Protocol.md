# Part03：Kernel 运行恢复协议

## 1. 目标

定义 Kernel 在暂停、异常、Snapshot Restore、输入重放和降级恢复时的唯一状态转换，避免多个模块各自恢复造成时间、队列和事实分叉。

## 2. 状态机

```text
Created → Ready → Running → Pausing → Paused
Running → Recovering → Replaying → Running
Running → Degraded → Recovering
Recovering → Failed
Paused → Sealed
```

## 3. 接口

```csharp
public interface IKernelRecoveryCoordinator
{
    ValueTask<RecoveryResult> RecoverAsync(
        RecoveryRequest request,
        CancellationToken cancellationToken);
}

public sealed record RecoveryRequest(
    Guid RunId,
    Guid SnapshotId,
    long TargetEventSequence,
    RecoveryMode Mode,
    Guid CorrelationId);
```

## 4. 恢复顺序

1. 停止接受新的外部输入。
2. 验证 Snapshot Manifest、Hash、RunId 和 SchemaVersion。
3. 恢复 Event Cursor、Input Cursor、ClockEpoch。
4. 恢复 World/Device/Lock/Occupancy 事实。
5. 恢复 PLC Process Image 和版本。
6. 恢复 Scheduler Queue，按确定性排序重新装载。
7. 从 Snapshot 之后重放已记录 InputRecord。
8. 生成 RecoveryCompleted 事件后恢复 Running。

## 5. 故障策略

- Snapshot Hash 不匹配：`KERNEL-RECOVERY-422`，禁止继续。
- Event Sequence 缺口：进入 Failed，要求人工选择 Full Replay。
- 未完成锁租约：按 LeaseUntilTick 判断接管或释放，禁止静默保留。
- 外部输入重复：按 `(RunId, ExternalSequence, ClockEpoch)` 去重。

## 6. 验收项

1. 恢复后 SimTick 不回退。
2. 恢复后 EventSequence 单调递增。
3. 相同 Snapshot + InputRecord 得到相同 ResultHash。
4. Recovery 期间不得产生对外完成事件。
5. 失败恢复必须保留完整诊断上下文。
