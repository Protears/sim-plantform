# Part 10：设备运行事件契约

## 1. 事件分类

| 类别 | 事件 | 语义 | 事实来源 |
|---|---|---|---|
| Intent | `DeviceCommandAccepted` | 命令已通过准入 | Command Store |
| State | `DeviceExecutionChanged` | 设备执行状态发生变化 | Device Runtime |
| Transfer | `CargoTransferCommitted` | 位置事实已提交 | World Model |
| Diagnostic | `DeviceExecutionFaulted` | 设备执行故障 | Device Runtime |
| Recovery | `DeviceRuntimeRecovered` | 运行态恢复完成 | Recovery Coordinator |

## 2. 统一 Envelope

```csharp
public sealed record RuntimeEventEnvelope<T>(
    Guid EventId,
    Guid RunId,
    long SimTick,
    long Sequence,
    string EventType,
    string SchemaVersion,
    string SourceType,
    Guid SourceId,
    Guid CorrelationId,
    Guid? CausationId,
    T Payload);

public sealed record DeviceExecutionChanged(
    Guid CommandId,
    Guid DeviceId,
    string PreviousStatus,
    string CurrentStatus,
    long DeviceVersion,
    Guid? TransferId,
    string? ErrorCode);

public sealed record CargoTransferCommitted(
    Guid TransferId,
    Guid CargoId,
    Guid? FromPlaceId,
    Guid ToPlaceId,
    long OccupancyVersion,
    string StateHash);
```

## 3. 发布顺序与幂等

- `DeviceCommandAccepted` 必须先于任何执行状态事件。
- `CargoTransferCommitted` 必须晚于 Occupancy Commit，不能由设备控制器直接发布。
- Sequence 由 Part10 Event Store 分配；同一 SimTick 使用固定优先级：Intent → State → Transfer → Diagnostic。
- 消费者以 `(RunId, EventId)` 幂等；状态投影以 `(RunId, AggregateId, Sequence)` 拒绝回退版本。

## 4. Outbox 规则

Command、Transfer、故障事实和 Outbox 在同一提交边界写入。发布失败只增加 attempt，不回滚已提交事实。达到重试上限后进入 `DeadLetter`，产生 `EVENT-OUTBOX-503`，但不得重复执行设备动作。

## 5. 验收

- 任意事件可通过 EventId 追溯到 CommandId/TransferId。
- 重复投递不得二次改变 DeviceVersion 或 OccupancyVersion。
- 事件重放后投影 Hash 与首次运行一致。
