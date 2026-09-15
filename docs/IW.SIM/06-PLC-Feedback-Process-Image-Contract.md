# Part 06：设备状态回写 PLC 过程映像契约

## 1. 边界

Device Runtime 产生设备事实，Signal/IO Runtime 负责将事实映射为逻辑信号，PLC Runtime 只在下一个扫描周期的 InputCommit 阶段读取。禁止设备运行时直接写 PLC 内存。

## 2. 核心接口

```csharp
public interface IPlcFeedbackPublisher
{
    FeedbackPublishResult Publish(DeviceStateSnapshot snapshot, PlcCycleContext context);
}

public sealed record DeviceStateSnapshot(
    Guid DeviceId,
    long DeviceVersion,
    DeviceRunState RunState,
    IReadOnlyDictionary<string, IoValue> Signals,
    long OccupancyVersion);
```

## 3. 提交规则

1. 设备状态只在 Kernel 上下文内生成快照。
2. Feedback Publisher 先完成信号类型、Quality、地址和版本校验。
3. 同一 `CycleSequence` 只允许一次 InputImageCommit。
4. 失败时整批回滚，不产生部分反馈。
5. `OccupancyVersion` 必须随完成类信号一并传递，避免“传感器到位但位置事实未提交”。

## 4. 状态映射

| Device Runtime 事实 | PLC 逻辑信号 | Quality |
|---|---|---|
| Ready | DeviceReady | Good |
| Executing | DeviceBusy | Good |
| Faulted | DeviceFault | Bad/Uncertain |
| Transfer Committed | CargoAtTarget | Good |
| Recovery | DeviceRecovering | Uncertain |

## 5. 错误码

- `PLC-FB-409`：反馈 CycleSequence 重复
- `PLC-FB-422`：信号类型或地址不匹配
- `PLC-FB-423`：OccupancyVersion 缺失
- `PLC-FB-503`：过程映像提交不可用

## 6. 验收

- 设备状态变化只能在下一周期被 PLC 观察到。
- Transfer Commit 失败时不得发布 CargoAtTarget=true。
- 同一周期反馈重复提交必须幂等。
- Quality=Bad 时，安全相关反馈按 FailSafe 映射。
