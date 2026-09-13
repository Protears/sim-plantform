# Part 10：设备命令、资源锁与接驳数据 Schema

## 1. 目标与数据责任

本文件把 Part05 Device Runtime、Part02 World Model、Part10 Data 的运行事实落到 PostgreSQL。数据库只保存已确认的事实和可靠投递状态，不承担设备算法、仿真时间推进或 PLC 扫描决策。

- `device_command`：命令意图、幂等键、生命周期和结果引用。
- `device_resource_lock`：运行时锁租约及恢复依据。
- `occupancy_transfer`：货载从源位置到目标位置的 Transfer 事实。
- `simulation_event`：全局有序事件事实，Sequence 由 Event Store 分配。
- `outbox_message`：跨模块可靠投递，不替代业务事实。

## 2. PostgreSQL Schema

```sql
create table device_command (
  run_id uuid not null,
  command_id uuid not null,
  device_id uuid not null,
  cargo_id uuid,
  command_type varchar(64) not null,
  request_hash varchar(128) not null,
  status varchar(24) not null,
  expected_device_version bigint,
  actual_device_version bigint,
  target_sim_tick bigint not null,
  deadline_sim_tick bigint,
  correlation_id uuid not null,
  causation_id uuid,
  result_code varchar(64),
  result_payload jsonb,
  accepted_at timestamptz not null default now(),
  completed_at timestamptz,
  state_version bigint not null default 0,
  primary key (run_id, command_id),
  constraint ck_device_command_status check (status in
    ('Accepted','Queued','Executing','Succeeded','Failed','CancelRequested','Cancelled','Expired')),
  constraint ck_device_command_deadline check
    (deadline_sim_tick is null or deadline_sim_tick >= target_sim_tick)
);

create unique index uq_device_command_request
  on device_command(run_id, command_id, request_hash);
create index ix_device_command_device_tick
  on device_command(run_id, device_id, target_sim_tick, status);
create index ix_device_command_correlation
  on device_command(run_id, correlation_id);

create table device_resource_lock (
  run_id uuid not null,
  lock_id uuid not null,
  resource_type varchar(32) not null,
  resource_id uuid not null,
  command_id uuid not null,
  acquired_sim_tick bigint not null,
  lease_until_sim_tick bigint not null,
  status varchar(16) not null,
  state_version bigint not null default 0,
  primary key (run_id, lock_id),
  unique (run_id, resource_type, resource_id),
  constraint ck_resource_lock_status check (status in
    ('Requested','Held','Renewing','Released','Expired'))
);

create index ix_resource_lock_command on device_resource_lock(run_id, command_id, status);

create table occupancy_transfer (
  run_id uuid not null,
  transfer_id uuid not null,
  cargo_id uuid not null,
  from_place_id uuid,
  to_place_id uuid not null,
  device_id uuid not null,
  status varchar(20) not null,
  expected_occupancy_version bigint not null,
  committed_occupancy_version bigint,
  state_hash varchar(128),
  correlation_id uuid not null,
  created_sim_tick bigint not null,
  committed_sim_tick bigint,
  primary key (run_id, transfer_id),
  unique (run_id, cargo_id, status),
  constraint ck_transfer_status check (status in
    ('IntentCreated','Claimed','Prepared','Committed','Rejected','Compensating','Compensated'))
);

create unique index uq_active_transfer_by_cargo
  on occupancy_transfer(run_id, cargo_id)
  where status in ('IntentCreated','Claimed','Prepared','Compensating');
create index ix_transfer_to_place on occupancy_transfer(run_id, to_place_id, status);

create table outbox_message (
  message_id uuid primary key,
  run_id uuid not null,
  aggregate_type varchar(64) not null,
  aggregate_id uuid not null,
  event_type varchar(128) not null,
  payload jsonb not null,
  status varchar(16) not null default 'Pending',
  attempt_count integer not null default 0,
  next_attempt_at timestamptz not null default now(),
  last_error varchar(512),
  created_at timestamptz not null default now(),
  published_at timestamptz,
  unique(run_id, message_id)
);

create index ix_outbox_pending
  on outbox_message(status, next_attempt_at)
  where status = 'Pending';
```

## 3. 事务边界与幂等

1. 命令准入事务：写入 `device_command`，必要时写入 `outbox_message`，成功后才允许发布 `DeviceCommandAccepted`。
2. Claim 事务：写入 `occupancy_transfer` 的 `Claimed` 状态并校验目标位置版本。
3. Commit 事务：更新 World Occupancy、Transfer、Outbox；不得只更新 Transfer 而不更新位置事实。
4. 消费幂等：`(run_id, event_id)` 由 Part10 Event Store 保证；业务表使用 `command_id`、`transfer_id` 作为幂等键。
5. 设备命令重试必须复用原 `command_id`；`request_hash` 不一致时拒绝。

## 4. 恢复与保留

Snapshot 必须包含活动命令、未释放锁、活动 Transfer、各自的 `state_version`。恢复时先恢复事实表，再重建内存索引；锁租约已过期的记录进入 `Expired`，不得自动恢复为 `Held`。运行期间 Hot，封存后转 Warm；Outbox 已发布记录保留至 Run 归档完成。

## 5. 验收门槛

- 100 个并发设备下命令准入无重复 CommandId。
- 同一目标位置的并发 Transfer 只有一个 Commit 成功。
- 进程中断恢复后，未完成命令和锁状态与 Snapshot 一致。
- Replay 后 Device/Occupancy/Result Hash 与原运行一致。
