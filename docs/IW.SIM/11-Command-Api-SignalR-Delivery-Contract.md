# Part 11：命令 API 与 SignalR 事实投递契约

## 1. API 边界

REST 只提交命令意图和查询状态；命令执行由 Application 编排并统一进入 Command Admission。客户端不得调用 Device Runtime 实现类。

## 2. API 契约

- `POST /api/simulation-runs/{runId}/device-commands`
- `GET /api/simulation-runs/{runId}/device-commands/{commandId}`
- `POST /api/simulation-runs/{runId}/device-commands/{commandId}/cancel`

请求必须包含：`commandId、deviceId、operation、payload、requestHash、expectedDeviceVersion、clientRequestId`。

响应语义：

- `202 Accepted`：已进入准入流程
- `200 OK`：查询现状
- `409 Conflict`：版本/Run 状态冲突
- `422 Unprocessable Entity`：RequestHash 或契约错误
- `423 Locked`：资源或 Run 被锁定
- `503 Service Unavailable`：运行时不可用

## 3. SignalR

Hub：`/hubs/simulation-runs`。客户端按 `Sequence` 处理事件，维护 `lastSequence`；断线重连后调用补发接口，不依赖客户端本地缓存恢复事实。

## 4. C# Contract

```csharp
public sealed record DeviceCommandAcceptedDto(
    Guid RunId, Guid CommandId, Guid DeviceId,
    long Sequence, long SimTick, string Status,
    string CorrelationId, int SchemaVersion);

public interface IRunEventReplayService
{
    IAsyncEnumerable<RuntimeEventEnvelope> ReplayAsync(
        Guid runId, long fromSequence, CancellationToken ct);
}
```

## 5. 验收

1. Sealed Run 的 POST 命令返回 409。
2. 相同 CommandId 重复提交返回同一结果。
3. SignalR 重复事件按 Sequence 被客户端丢弃。
4. fromSequence 补发后事件顺序连续。
5. API 日志可通过 CorrelationId 关联到 EventId 和 RunId。
