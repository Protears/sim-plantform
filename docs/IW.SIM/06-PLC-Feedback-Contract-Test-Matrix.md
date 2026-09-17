# Part06：PLC 反馈契约测试矩阵

## 1. 目标

验证 Device Runtime、Occupancy Transfer、Signal/IO Runtime 与 PLC Process Image 之间的反馈语义，不允许设备内部状态直接伪造完成信号。

## 2. 反馈契约

```csharp
public sealed record PlcFeedbackFrame(
    Guid RunId,
    Guid DeviceId,
    long CycleSequence,
    long ProcessImageVersion,
    long? OccupancyVersion,
    SignalQuality Quality,
    IReadOnlyDictionary<string, SignalValue> Signals,
    Guid CorrelationId);

public interface IPlcFeedbackPublisher
{
    ValueTask<FeedbackPublishResult> PublishAsync(
        PlcFeedbackFrame frame,
        CancellationToken cancellationToken);
}
```

## 3. 关键规则

1. `CargoAtDestination=true` 只有在 `CargoTransferCommitted` 已提交后才能发布。
2. 旧 `ProcessImageVersion` 不得覆盖新版本。
3. `Quality=Bad` 时 Critical 完成信号必须回落到 FailSafeValue。
4. 同一 `CycleSequence` 只能提交一个 Committed Feedback。
5. 反馈发布失败不得修改 Device/Occupancy 事实。

## 4. 测试矩阵

| ID | 场景 | 预期 |
|---|---|---|
| FB-001 | 正常 Transfer Commit | 下一 PLC 周期可见完成反馈 |
| FB-002 | Commit 冲突 | 不产生完成信号，返回 WORLD-OCC-409 |
| FB-003 | 旧 ImageVersion | 拒绝覆盖并记录 PLC-FB-409 |
| FB-004 | Bad Quality | Critical 输出为安全值 |
| FB-005 | 同 Cycle 重复发布 | 第二次幂等返回 |
| FB-006 | Feedback 通道不可用 | 事实保留，进入 Degraded |
| FB-007 | 重连后旧 Epoch | 丢弃旧反馈 |
| FB-008 | Snapshot Restore | ProcessImageVersion 不回退 |
| FB-009 | Replay | Feedback Hash 一致 |
| FB-010 | 100 PLC 并发 | 周期抖动在门槛内 |

## 5. 验收门槛

- 关键反馈丢失率为 0。
- 旧版本覆盖次数为 0。
- 72 小时运行无反馈序列断裂。
- P95 反馈提交延迟不超过一个仿真周期。
