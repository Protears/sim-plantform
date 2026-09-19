# Part 11：Trace 查询与运维 API 契约

## 1. 目标

提供统一的 Run、Command、Transfer、Recovery、Audit 和 Trace 查询入口；查询只读，不改变运行事实；所有分页和补发均使用 EventSequence 或明确的时间范围，禁止使用不稳定的 offset 分页。

## 2. REST API

```http
GET /api/runs/{runId}
GET /api/runs/{runId}/events?afterSequence=120&limit=200
GET /api/runs/{runId}/trace/{correlationId}
GET /api/runs/{runId}/audit?afterSequence=120&limit=100
POST /api/runs/{runId}/operations
POST /api/runs/{runId}/replays
```

## 3. DTO

```csharp
public sealed record TraceQueryResult(
    Guid RunId,
    string CorrelationId,
    IReadOnlyList<TraceNode> Nodes,
    bool Complete,
    string? MissingReason);

public sealed record TraceNode(
    string NodeType,
    string NodeId,
    long? Sequence,
    long? SimTick,
    string State,
    string Hash);

public sealed record EventPage<T>(
    IReadOnlyList<T> Items,
    long? NextSequence,
    bool HasGap,
    string? GapReason);
```

## 4. 查询规则

1. `afterSequence` 是排他游标；返回数据必须按 `EventSequence` 升序。
2. 检测到 Sequence 缺口时返回 `hasGap=true`，客户端必须进入 Resync/Replay，不得继续拼接为完整事实。
3. Trace 查询必须同时校验 `CommandId → TransferId → OccupancyVersion → ProcessImageVersion` 的单调性。
4. 查询接口不能从 Telemetry 推断 Command/Occupancy 完成状态。
5. Replay 查询返回独立 `ReplayRunId` 和结果 Hash，不得覆盖源 Run 的结果。

## 5. SignalR 规则

Hub：`/hubs/simulation-runs`

```csharp
public interface IRunEventClient
{
    Task EventAppended(EventEnvelope message);
    Task ResyncRequired(ResyncRequiredMessage message);
    Task RunStateChanged(RunStateChangedMessage message);
}
```

客户端必须：

- 保存最后确认的 EventSequence；
- 断线后携带 `afterSequence` 重订阅；
- 按 EventId 和 EventSequence 双重去重；
- 收到 ResyncRequired 时暂停业务展示，直到补发完成。

## 6. 错误码

- `QUERY-001`：afterSequence 非法。
- `QUERY-002`：Sequence 缺口，需要 Replay/Resync。
- `QUERY-003`：Trace 关键节点缺失。
- `QUERY-004`：ReplayRunId 不属于当前查询上下文。

## 7. 直接工程任务

- 实现 EventSequence 查询索引和只读投影。
- 实现 Trace Query 聚合器和缺口检测。
- 实现 SignalR 重订阅、补发和 ResyncRequired。
- 增加查询、断线、缺口、Replay 隔离测试。
