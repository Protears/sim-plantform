# Part 10：World State Reconciliation Schema

## 1. PostgreSQL Schema

```sql
create table world_reconciliation_record (
    reconciliation_id uuid primary key,
    run_id uuid not null,
    device_id uuid not null,
    sim_tick bigint not null,
    drift_code varchar(64) not null,
    severity varchar(16) not null,
    observed_payload jsonb not null,
    expected_payload jsonb not null,
    layout_snapshot_id uuid not null,
    binding_snapshot_id uuid not null,
    occupancy_version bigint not null,
    process_image_version bigint not null,
    status varchar(24) not null,
    operation_id uuid,
    created_at timestamptz not null,
    resolved_at timestamptz,
    resolution_hash varchar(128)
);

create index ix_world_reconciliation_run_status
    on world_reconciliation_record(run_id, status, created_at);

create unique index ux_world_reconciliation_active_device
    on world_reconciliation_record(run_id, device_id)
    where status in ('Detected','Degraded','Escalated');
```

## 2. 数据责任

- 记录只追加，不物理删除。
- `observed_payload` 保存原始观测，`expected_payload` 保存权威事实摘要。
- 人工修复必须通过 OperationId，并写入审计和 resolution_hash。
- Reconciliation 记录不能直接修改 Occupancy；必须调用 `IOccupancyAuthority`。

## 3. 迁移规则

新增字段遵循 Expand → Backfill → Switch → Contract。`status` 扩展必须兼容旧 Worker 读取。记录保留策略不得删除未解决或与 Run 证据包关联的数据。

## 4. 查询索引

常用查询：Run 内未解决冲突、设备近 24 小时漂移、按 SimTick 关联 EventSequence、按 OperationId 查询人工修复。
