# Part 11：Run Event Query 与 Replay Contract v2

## 1. 目标

统一 REST、SignalR、后台投影和 Replay 的事件游标。客户端不使用时间戳分页；唯一顺序游标为 `EventSequence`。

## 2. DTO

```csharp
public sealed record RunEventQuery(
    Guid RunId,
    long? AfterSequence,
    int Limit,
    IReadOnlySet<string>? EventTypes);

public sealed record RunEventPage(
    Guid RunId,
    long? FirstSequence,
    long? LastSequence,
    bool HasMore,
    IReadOnlyList<RuntimeEventEnvelope> Items);

public sealed record ReplayRequest(
    Guid SourceRunId,
    long FromSequence,
    long? ToSequence,
    Guid ReplayRunId);
```

## 3. API

```text
GET  /api/simulation-runs/{runId}/events?afterSequence=100&limit=200
POST /api/simulation-runs/{runId}/replays
GET  /api/simulation-runs/{runId}/replays/{replayRunId}
```

规则：

- `afterSequence` 为排他游标。
- `limit` 最大 1000，超限返回 400。
- `ReplayRunId` 必须新建，禁止复用源 RunId。
- Source Run Sealed 后允许 Replay，但不允许写入源事实。

## 4. SignalR

Hub：`/hubs/simulation-runs`

```text
SubscribeRun(runId, afterSequence)
UnsubscribeRun(runId)
ReplayMissed(runId, afterSequence)
```

服务端必须：

1. 先按游标补发已提交事件。
2. 再订阅实时事件。
3. 按 Sequence 去重。
4. 断线重连后从客户端最后确认的 Sequence 补发。

## 5. 一致性规则

- 只发布已提交 Event Store 事件。
- Outbox 未 Published 的事件不可通过 SignalR 提前可见。
- 投影落后时查询源 Event Store，不返回未解释的空洞。
- EventSequence 出现缺口时客户端进入 ResyncRequired。

## 6. 测试矩阵

| 场景 | 预期 |
|---|---|
| afterSequence=0 | 从首个事件开始 |
| 游标不存在 | 返回最近有效游标并标记 ResyncRequired |
| SignalR 断线 | 从最后 Ack Sequence 补发 |
| 重复事件 | 客户端只处理一次 |
| Replay 中 | 不影响源 Run |
| Outbox 延迟 | 不提前推送 |
| Sequence 缺口 | 阻断继续消费 |
