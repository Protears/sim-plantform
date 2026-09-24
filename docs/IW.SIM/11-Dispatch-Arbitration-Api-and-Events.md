# Dispatch Arbitration API 与事件契约

## 1. API

### POST `/api/runs/{runId}/dispatch-arbitrations`

请求：

```json
{
  "taskId": "uuid",
  "subTaskId": "uuid",
  "cargoId": "uuid",
  "routeId": "uuid",
  "priority": 50,
  "deadlineAt": "2026-09-24T10:00:00Z",
  "worldVersion": "wv-17",
  "planHash": "sha256:...",
  "operationId": "uuid"
}
```

响应：`202 Accepted`，返回 `arbitrationId` 和 `queueRank`。

### POST `/api/runs/{runId}/dispatch-arbitrations/{arbitrationId}/preempt`

仅允许状态为 `Granted` 或 `WaitingResource`。`Executing` 返回 `423`。

## 2. Event Envelope

```json
{
  "eventType": "DispatchGranted",
  "schemaVersion": 1,
  "runId": "uuid",
  "eventId": "uuid",
  "eventSequence": 1182,
  "correlationId": "uuid",
  "causationId": "uuid",
  "arbitrationId": "uuid",
  "taskId": "uuid",
  "routeId": "uuid",
  "reservationId": "uuid",
  "stateHash": "sha256:..."
}
```

事件：`DispatchQueued`、`DispatchGranted`、`DispatchRejected`、`DeadlockDetected`、`DeadlockVictimSelected`、`DispatchPreempted`、`DispatchReleased`。

## 3. SignalR

`SubscribeDispatch(runId, afterSequence)` 必须沿用 EventSequence 补发规则；缺口返回 `ResyncRequired`。

## 4. 错误映射

- `409`：WorldVersion/状态冲突
- `423`：正在执行，不允许抢占
- `422`：PlanHash/RouteId/TaskId 不匹配
- `429`：Deadline 已错过
- `503`：仲裁服务不可用
