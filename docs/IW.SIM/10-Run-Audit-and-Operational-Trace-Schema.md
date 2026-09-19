# Part 10：Run Audit 与 Operational Trace Schema

## 1. 目标

本文件定义配置冻结、命令准入、Recovery、Replay、手工干预和故障升级的不可抵赖审计链。它与 Event Store、Telemetry、Outbox 分工不同：Audit 记录“谁在什么上下文下做了什么运维动作”，Trace 记录一次业务闭环跨模块的关联键。

## 2. 核心实体

```csharp
public sealed record OperationalAuditEntry(
    Guid AuditId,
    Guid RunId,
    string ActionType,
    string ActorType,
    string ActorId,
    string CorrelationId,
    string? CausationId,
    long SimTick,
    long EventSequence,
    string BeforeHash,
    string AfterHash,
    string Outcome,
    string SchemaVersion,
    DateTimeOffset RecordedAt);

public sealed record RuntimeTraceContext(
    Guid RunId,
    long CycleSequence,
    Guid? CommandId,
    Guid? TransferId,
    long? OccupancyVersion,
    long? ProcessImageVersion,
    long? EventSequence,
    string CorrelationId,
    string? CausationId);
```

## 3. 审计动作分类

| ActionType | 例子 | 是否必须审计 |
|---|---|---|
| RunCreated | 创建 Run | 是 |
| ProfileBound | 绑定配置 | 是 |
| CommandSubmitted | 外部提交命令 | 是 |
| CommandCancelled | 取消命令 | 是 |
| RecoveryStarted | 开始恢复 | 是 |
| RecoveryCompleted | 恢复完成 | 是 |
| ReplayStarted | 开始 Replay | 是 |
| ManualOverride | 人工干预 | 是 |
| ConfigRejected | 配置拒绝 | 是 |
| SafetyFallback | 安全回退 | 是 |

## 4. PostgreSQL Schema

```sql
create table simulation_operational_audit (
    audit_id uuid primary key,
    run_id uuid not null,
    action_type varchar(64) not null,
    actor_type varchar(32) not null,
    actor_id varchar(128) not null,
    correlation_id uuid not null,
    causation_id uuid null,
    sim_tick bigint not null,
    event_sequence bigint not null,
    before_hash varchar(128) not null,
    after_hash varchar(128) not null,
    outcome varchar(32) not null,
    schema_version int not null,
    recorded_at timestamptz not null,
    payload jsonb not null
);
create index ix_operational_audit_run_sequence
    on simulation_operational_audit(run_id, event_sequence);
create index ix_operational_audit_correlation
    on simulation_operational_audit(correlation_id);
```

## 5. 规则

1. Audit 必须在对应事实提交后写入，不得先写成功审计再提交事实。
2. Recovery/Replay 的审计必须携带 `SnapshotId` 或 `ReplayRunId`。
3. `before_hash`、`after_hash` 不允许为空；无状态变化时使用同一 Hash。
4. Audit 只追加，不更新、不物理删除。
5. EventSequence 来自 Event Store；Audit 不得自行分配全局业务序列。
6. 个人信息和敏感连接参数不得进入 payload，使用脱敏摘要。

## 6. Trace 闭环

```text
CycleSequence
  -> CommandId
  -> TransferId
  -> OccupancyVersion
  -> ProcessImageVersion
  -> EventSequence
  -> AuditId
```

任何设备完成、故障、恢复和人工干预页面必须能够按 `CorrelationId` 反查上述链路；缺失关键节点时返回 `TRACE-INCOMPLETE`，不得静默显示“成功”。

## 7. 错误码

- `AUDIT-001`：事实已提交但审计写入失败，Run 进入 Degraded。
- `AUDIT-002`：发现重复 AuditId，按幂等处理。
- `TRACE-001`：Trace 关键节点缺失。
- `TRACE-002`：Trace 的版本或 Sequence 回退。

## 8. 直接工程任务

- 实现 `IOperationalAuditStore` 和追加写入策略。
- 为 Recovery、Replay、ManualOverride 添加审计中间件。
- 建立 Trace 查询 API 和缺失节点告警。
- 增加 Audit Append、Trace Completeness、Sequence Monotonicity 测试。
