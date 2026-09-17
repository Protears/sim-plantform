# Part11：Run 事件查询与回放契约

## 1. 目标

为 Application、UI、诊断和自动化测试提供统一的 Run 事件查询、Sequence 补发和 Replay 入口，不允许客户端直接读取底层事件表推断运行态。

## 2. API

```http
GET /api/simulation-runs/{runId}/events?afterSequence=120&limit=500
POST /api/simulation-runs/{runId}/replay
GET /api/simulation-runs/{runId}/replay/{replayRunId}
```

```csharp
public sealed record RunEventPage(
    Guid RunId,
    long FromSequence,
    long ToSequence,
    bool HasMore,
    IReadOnlyList<RuntimeEventEnvelope> Events);

public interface IRunEventReplayService
{
    ValueTask<RunEventPage> ReadAsync(Guid runId, long afterSequence, int limit, CancellationToken ct);
    ValueTask<ReplayRunResult> StartAsync(ReplayRequest request, CancellationToken ct);
}
```

## 3. 规则

- `afterSequence` 为排他游标，客户端按 Sequence 去重。
- 返回页必须保证同一 Run 内按 Sequence 升序。
- Replay 生成新的 `ReplayRunId`，不得写回原 Run。
- Sealed Run 允许查询和 Replay，禁止命令写入。
- 事件 Payload 版本不兼容时返回 `EVENT-SCHEMA-409`，不得静默转换。

## 4. SignalR 补发

Hub 方法：`SubscribeRun(runId, afterSequence)`。连接建立后先补发缺口，再订阅实时消息；补发完成前不发送实时增量，避免顺序交叉。

## 5. 可验证项

1. Sequence 缺口可补发。
2. 重连后不重复消费业务事件。
3. Replay 与原 Run 的 Event/State/Result Hash 可比较。
4. 未授权 Run 查询返回 403，不泄漏事件 Payload。
5. 大页查询强制上限 1000，防止阻塞运行时。
