# Part 10：EF Core 数据访问契约与事务实现

## 1. 目标

本文件把设备命令、资源锁、Occupancy Transfer、Outbox 与 Simulation Event 的数据库设计转化为 EF Core 10 实施规则。Data 层只实现 Store，不向上泄露 `DbContext`、查询表达式或 PostgreSQL 细节。

## 2. DbContext 边界

```csharp
public interface ISimulationUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
    Task ExecuteInTransactionAsync(
        Func<CancellationToken, Task> action,
        CancellationToken cancellationToken);
}

public interface IDeviceCommandStore
{
    Task<CommandRecord?> FindAsync(Guid runId, Guid commandId, CancellationToken ct);
    Task InsertAcceptedAsync(CommandRecord record, CancellationToken ct);
    Task<bool> TryTransitionAsync(Guid runId, Guid commandId,
        string fromStatus, string toStatus, long expectedVersion,
        CancellationToken ct);
}
```

`DbContext` 仅存在 `Logistics.Simulation.Data`。Domain/Application 依赖上述接口，不依赖 EF 类型。

## 3. 映射约束

- 表名和列名统一 snake_case。
- UUID 采用 PostgreSQL `uuid`；SimTick、Version、Sequence 采用 `bigint`。
- JSON 契约使用 `jsonb`，同时保存 `schema_version`。
- 所有运行事实表必须拥有 `run_id`，查询默认必须带 Run 过滤。
- 状态转换通过 `state_version` 乐观并发，不允许无条件更新。
- Active Transfer、Active Lock 等部分唯一索引必须在迁移脚本中显式创建。

## 4. 事务边界

### 命令准入事务

同一事务写入 `device_command` 与 `outbox_message`；只有提交成功后才可将 Accepted 事件交给发布器。

### Occupancy Commit 事务

在同一事务内：锁定来源与目标位置、校验版本、更新 Occupancy、更新 `occupancy_transfer`、写入 `simulation_event`/Outbox。任何一步失败全部回滚。

### 事件消费事务

Inbox 去重记录、业务状态更新、消费结果必须在同一事务内完成。重复事件返回已存在的消费结果，不再次触发设备动作。

## 5. 查询与索引

- 运行态查询优先使用 `(run_id, status, target_sim_tick)`。
- 设备历史查询使用 `(run_id, device_id, created_sim_tick desc)`。
- Outbox 使用 Pending 部分索引和 `next_attempt_at` 排序。
- 归档查询与在线查询分离，禁止在线请求扫描已封存 Run 的全部事件。

## 6. 迁移与回滚

迁移顺序：新增表/索引 → 双写字段 → 切换读取 → 删除旧字段。禁止在单次迁移中直接删除仍被旧运行时读取的字段。迁移必须具备向前和向后兼容说明，并在 CI 中执行空库迁移和从最近生产快照迁移。

## 7. 验收项

1. 并发命令准入不会生成重复 `(run_id, command_id)`。
2. Occupancy Commit 冲突时设备命令状态不会错误变为 Completed。
3. Outbox 重试不会重复改变业务状态。
4. 版本冲突能被稳定映射为 `409` 领域错误。
5. Snapshot 恢复后所有活动记录可按 RunId 重建。
