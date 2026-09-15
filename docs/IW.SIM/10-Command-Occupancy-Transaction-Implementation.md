# Part 10：命令、Transfer 与 Outbox 事务实施契约

## 1. 事务原则

业务结果以数据库事务提交为准；事件发布是提交后的可靠投递，不得反向决定业务是否成功。Kernel 不等待网络或消息中间件确认。

## 2. 三类事务

### T1 命令准入

`CommandIdempotency Insert → DeviceCommand Insert → CommandAccepted Outbox Insert → Commit`。
唯一键：`(run_id, command_id)`。

### T2 Transfer Commit

`From/To/Cargo 行锁 → ExpectedOccupancyVersion 校验 → Occupancy 更新 → Transfer 状态 Committed → CargoTransferCommitted Outbox → Commit`。

### T3 消费幂等

`Inbox Insert(event_id) → Projection/Handler → Inbox Processed → Commit`。重复 EventId 直接返回已处理结果。

## 3. C# 端口

```csharp
public interface IBusinessTransaction
{
    Task<TResult> ExecuteAsync<TResult>(
        Func<ITransactionContext, CancellationToken, Task<TResult>> action,
        CancellationToken cancellationToken);
}

public interface IOutboxStore
{
    Task AddAsync(OutboxMessage message, CancellationToken ct);
    Task<IReadOnlyList<OutboxMessage>> LeaseBatchAsync(int size, DateTimeOffset now, CancellationToken ct);
    Task MarkPublishedAsync(Guid messageId, DateTimeOffset publishedAt, CancellationToken ct);
}
```

## 4. 失败语义

- DB 回滚：不得发布 Accepted/Committed。
- Outbox 发布失败：业务状态保持成功，进入重试。
- Handler 失败：Inbox 保持 Pending，允许重试；超过阈值进入 DeadLetter。
- Kernel 只接收已提交事实，不读取未提交缓存作为恢复依据。

## 5. 观测字段

所有事务日志必须包含：`RunId、CommandId/TransferId、EventId、CorrelationId、SimTick、TransactionId、RetryCount`。

## 6. 验收项

1. 命令插入成功但 Outbox 插入失败时整体回滚。
2. Transfer Commit 成功后必有一条 Pending/Published Outbox。
3. 同一 EventId 重投不得二次改变 Occupancy。
4. Outbox 租约超时可被其他 Worker 接管，但不重复执行业务动作。
5. 72 小时运行无孤儿 Pending 事务记录。
