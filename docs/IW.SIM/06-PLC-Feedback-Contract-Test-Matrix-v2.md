# Part 06：PLC Feedback Contract Test Matrix v2

## 1. 反馈帧

```csharp
public sealed record PlcFeedbackFrame(
    Guid RunId,
    Guid DeviceId,
    long CycleSequence,
    long ProcessImageVersion,
    long? OccupancyVersion,
    string Quality,
    bool CargoAtDestination,
    string? CommandId,
    string? TransferId,
    string CorrelationId);
```

## 2. 测试矩阵

| 编号 | 场景 | 预期结果 |
|---|---|---|
| FB-001 | Transfer 未 Commit | CargoAtDestination=false |
| FB-002 | OccupancyVersion 缺失 | 拒绝完成类反馈 |
| FB-003 | 旧 ProcessImageVersion | 丢弃并审计 |
| FB-004 | 旧 ClockEpoch | 拒绝 |
| FB-005 | Quality=Bad | 关键完成信号进入 FailSafe |
| FB-006 | 同一 CycleSequence 重复 | 幂等，不重复提交 |
| FB-007 | CommandId 不存在 | 反馈转诊断事件 |
| FB-008 | TransferId 不匹配 | 反馈失败 |
| FB-009 | Outbox 未提交 | 不向 PLC/SignalR 发布 |
| FB-010 | 恢复后版本回退 | Run 进入 Failed |
| FB-011 | 设备状态与 Occupancy 不一致 | 触发 Reconciliation |
| FB-012 | 72 小时运行 | 无重复、无版本回退、无序列缺口 |

## 3. 验收指标

- 反馈提交延迟 P95 ≤ 1 个 SimTick。
- 关键反馈错误必须包含 RunId、CycleSequence、CommandId、TransferId、OccupancyVersion。
- 反馈和 Event Sequence 可双向追踪。
- 任何测试失败均阻断 PLC 联调发布。