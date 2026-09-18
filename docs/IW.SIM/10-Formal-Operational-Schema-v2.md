# Part 10：正式运行态 Schema v2

## 1. 目标

本文件将 Command、Transfer、ResourceLock、Event、Outbox、Snapshot 统一为可迁移、可并发控制、可恢复的数据基线。Schema 只存事实和投递状态，不存设备算法临时变量。

## 2. 事务事实边界

| 事务 | 必须原子提交 | 禁止包含 |
|---|---|---|
| T1 CommandAdmission | device_command、accepted event、accepted outbox | 设备动作执行 |
| T2 TransferCommit | occupancy_transfer、world occupancy、committed event、outbox | 网络发送 |
| T3 EventConsume | inbox_message、projection checkpoint、projection rows | 修改原始事实 |
| T4 Snapshot | snapshot metadata、snapshot payload、integrity hash | 未提交业务状态 |

## 3. PostgreSQL Schema

```sql
create table device_command (
  run_id uuid not null,
  command_id uuid not null,
  device_id uuid not null,
  command_type varchar(80) not null,
  request_hash char(64) not null,
  state varchar(32) not null,
  expected_device_version bigint,
  actual_device_version bigint,
  transfer_id uuid,
  payload jsonb not null,
  result jsonb,
  error_code varchar(64),
  state_version bigint not null default 0,
  created_sim_tick bigint not null,
  updated_sim_tick bigint not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  primary key (run_id, command_id)
);

create index ix_device_command_run_state on device_command(run_id, state, updated_sim_tick);
create unique index ux_device_command_active_device on device_command(run_id, device_id)
  where state in ('Accepted','Queued','Executing','TransferPending');

create table occupancy_transfer (
  run_id uuid not null,
  transfer_id uuid not null,
  command_id uuid not null,
  cargo_id uuid not null,
  from_place_id uuid,
  to_place_id uuid not null,
  state varchar(32) not null,
  expected_occupancy_version bigint not null,
  committed_occupancy_version bigint,
  state_hash char(64),
  payload jsonb not null,
  state_version bigint not null default 0,
  created_sim_tick bigint not null,
  updated_sim_tick bigint not null,
  primary key (run_id, transfer_id),
  unique (run_id, command_id)
);

create unique index ux_transfer_active_cargo on occupancy_transfer(run_id, cargo_id)
  where state in ('IntentCreated','Claimed','Prepared','Compensating');
create unique index ux_transfer_active_destination on occupancy_transfer(run_id, to_place_id)
  where state in ('Claimed','Prepared');

create table device_resource_lock (
  run_id uuid not null,
  lock_id uuid not null,
  resource_type varchar(32) not null,
  resource_id uuid not null,
  owner_command_id uuid not null,
  lease_until_tick bigint not null,
  state varchar(16) not null,
  state_version bigint not null default 0,
  primary key (run_id, lock_id)
);
create unique index ux_resource_lock_active on device_resource_lock(run_id, resource_type, resource_id)
  where state in ('Held','Renewing');

create table outbox_message (
  message_id uuid primary key,
  run_id uuid not null,
  aggregate_type varchar(64) not null,
  aggregate_id uuid not null,
  event_sequence bigint not null,
  event_type varchar(120) not null,
  schema_version int not null,
  payload jsonb not null,
  state varchar(16) not null default 'Pending',
  attempt_count int not null default 0,
  next_attempt_at timestamptz not null default now(),
  leased_until timestamptz,
  published_at timestamptz,
  last_error varchar(512),
  unique(run_id, event_sequence)
);
create index ix_outbox_dispatch on outbox_message(state, next_attempt_at, event_sequence);
```

## 4. 并发与幂等

- 所有更新必须带 `where state_version = @expected`，成功后 `state_version + 1`。
- 相同 `(run_id, command_id)`、`(run_id, transfer_id)`、`(run_id, event_sequence)` 重试必须返回原事实。
- RequestHash 冲突不允许覆盖原 Payload。
- Outbox Dispatcher 只能改变投递字段，不能改变业务事实字段。

## 5. 迁移规则

采用 Expand → Backfill → Switch → Contract。SchemaVersion 不兼容时 Runtime 不得进入 Running。大表索引必须使用并发创建，并提供回滚和观测指标。

## 6. 验收

- T1/T2 事务回滚后无 Accepted/Committed 外部可见事件。
- 并发 Claim 只能一个成功。
- Dispatcher 重试不得重复执行 Device Runtime。
- Snapshot 恢复后所有活动命令、锁、Transfer 与版本一致。