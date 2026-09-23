# Task 编排 API 与事件契约

## 1. REST API

### 创建任务
`POST /api/runs/{runId}/tasks`

```json
{
  "taskId": "uuid",
  "taskType": "MoveCargo",
  "cargoId": "uuid",
  "sourcePlaceId": "uuid",
  "targetPlaceId": "uuid",
  "priority": 50,
  "dependsOn": [],
  "expectedWorldVersion": 12,
  "requestHash": "sha256"
}
```

### 查询任务
`GET /api/runs/{runId}/tasks/{taskId}`

### 取消任务
`POST /api/runs/{runId}/tasks/{taskId}/cancel`

### 重新规划
`POST /api/runs/{runId}/tasks/{taskId}/replan`

## 2. HTTP 语义

- `202`：已接受，等待规划/执行。
- `409`：状态版本、WorldVersion、资源或 Cargo 冲突。
- `422`：请求结构、依赖图或能力约束不合法。
- `423`：Run Sealed 或任务被锁定。
- `503`：编排器或 Reservation 服务不可用。

## 3. 事件

```csharp
public sealed record TaskEventEnvelope<T>(
    Guid EventId,
    Guid RunId,
    Guid TaskId,
    string EventType,
    long SimTick,
    long EventSequence,
    string CorrelationId,
    string CausationId,
    string SchemaVersion,
    T Data);

public sealed record TaskCompleted(
    Guid TaskId,
    Guid CargoId,
    Guid TransferId,
    Guid RouteId,
    long OccupancyVersion,
    string StateHash);
```

事件类型：

- `TaskAccepted`
- `TaskPlanned`
- `TaskReserved`
- `TaskDispatched`
- `TaskExecutionFailed`
- `TaskCompensating`
- `TaskCompleted`
- `TaskCancelled`

## 4. SignalR

Hub：`/hubs/simulation-runs`

- `SubscribeTask(runId, taskId, afterSequence)`
- `TaskEventReceived(envelope)`
- `TaskResyncRequired(runId, taskId, expectedSequence)`

客户端必须按 `EventId` 去重，并按 `EventSequence` 检测缺口。

## 5. 完成语义

`TaskCompleted` 只能由 T-COMPLETE 事务写入；禁止由 API、WCS、PLC Mapper 或 Telemetry 直接发布。
