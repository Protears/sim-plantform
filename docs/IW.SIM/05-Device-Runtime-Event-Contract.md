# Part05 设备运行时事件契约

## 1. 事件分类

| 类型 | 示例 | 语义 |
|---|---|---|
| Intent | `DeviceCommandAccepted` | 接受执行意图 |
| State | `DeviceExecutionChanged` | 运行事实变化 |
| Transfer | `CargoTransferCompleted` | Occupancy 提交完成 |
| Diagnostic | `DeviceExecutionFaulted` | 故障与恢复线索 |

Intent 不代表成功；只有 State/Transfer 才能驱动下游事实判断。

## 2. 统一元数据

```csharp
public sealed record RuntimeEventEnvelope<T>(
    Guid EventId,
    Guid RunId,
    long Sequence,
    long SimTick,
    string EventType,
    string SourceType,
    Guid SourceId,
    string? CorrelationId,
    string? CausationId,
    int SchemaVersion,
    T Payload);
```

Sequence 由 Part10 Event Store 分配，业务模块不得自造全局 Sequence。`EventId` 在 `(RunId, EventId)` 范围内幂等。

## 3. 关键事件

### DeviceCommandAccepted
必须包含 `CommandId, DeviceId, CommandType, AdmissionSequence`。

### DeviceExecutionChanged
必须包含 `CommandId, PreviousStatus, CurrentStatus, DeviceVersion, SimTick`。

### CargoTransferCompleted
必须包含 `TransferId, CargoId, FromPlaceId, ToPlaceId, OccupancyVersion, StateHash`。

### DeviceExecutionFaulted
必须包含 `ErrorCode, Retryable, RecoveryAction, Context`。

## 4. 发布约束

- 事件发布顺序：Accepted → ExecutionChanged → TransferCompleted → Completed。
- 失败事件必须先于命令最终 Failed 状态。
- Event Publisher 不修改领域状态，只发布已提交事实。
- Outbox 记录与领域状态在同一事务边界内写入。

## 5. 验收

1. 任何 Completed 事件都能反查到对应的 OccupancyVersion。
2. 重复消费不会重复推进 DeviceVersion。
3. CorrelationId/CausationId 能串联 Application → Device → World → Event。
4. SchemaVersion 升级必须保留向后兼容反序列化策略。
