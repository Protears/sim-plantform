# Part05 设备执行事件契约

## 1. 事件分类

设备命令事件描述意图，设备状态事件描述事实，诊断事件描述异常。三类事件不可混用。

## 2. DTO/Event Contract

```csharp
public sealed record DeviceCommandAccepted(
    Guid EventId, Guid RunId, Guid CommandId, Guid DeviceId,
    long SimTick, string CommandType, long DeviceVersion);

public sealed record DeviceExecutionChanged(
    Guid EventId, Guid RunId, Guid CommandId, Guid DeviceId,
    long SimTick, string PreviousState, string CurrentState,
    string? FailureCode, long DeviceVersion);

public sealed record CargoTransferCompleted(
    Guid EventId, Guid RunId, Guid CommandId, Guid DeviceId,
    Guid CargoId, Guid? FromPlaceId, Guid ToPlaceId,
    long SimTick, long OccupancyVersion);
```

## 3. 事件元数据

所有事件必须包含 `EventId, RunId, SimTick, Sequence, SourceType, SourceId, CorrelationId, CausationId, SchemaVersion`。Sequence 由 Part10 Event Store 分配，业务模块不得自造全局 Sequence。

## 4. 幂等与顺序

消费端以 `(RunId, EventId)` 去重；同一 Command 的执行状态按 `DeviceVersion` 单调递增。发现版本回退时拒绝消费并产生 `EVENT-ORDER-409`。

## 5. 事务边界

设备状态变更、OccupancyCommit 和 Outbox 事件必须在同一运行时事务中完成；Telemetry 不参与事务。跨进程投递使用 Outbox，消费使用 Inbox。

## 6. 验收

- 事件重投不得重复改变设备状态。
- CommandAccepted 必须早于 ExecutionChanged。
- CargoTransferCompleted 必须携带有效 OccupancyVersion。
- 缺少 CorrelationId 的故障事件不得通过契约校验。
