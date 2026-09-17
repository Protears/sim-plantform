# Part10：运行态正式数据模型与迁移基线

## 1. 目标

将 Command、Transfer、Lock、Event、Outbox、Snapshot 的运行态数据收敛为可执行 PostgreSQL/EF Core 迁移基线，保证事务、幂等、恢复与 Replay 使用同一事实模型。

## 2. 表职责

| 表 | 权威事实 | 写入者 | 读取者 |
|---|---|---|---|
| `simulation_run` | Run 生命周期 | Application | 全部模块 |
| `device_command` | 命令状态与 RequestHash | Command Admission | Device Runtime/Application |
| `occupancy_transfer` | 货载位置转移状态 | World Model | Device Runtime/Replay |
| `device_resource_lock` | 资源租约 | Lock Manager | Device Runtime/Recovery |
| `simulation_event` | 全局有序事件 | Event Store | Replay/Projection |
| `outbox_message` | 已提交事实的待发布消息 | 同一业务事务 | Dispatcher |
| `simulation_snapshot` | 可恢复一致性点 | Snapshot Coordinator | Recovery |

## 3. PostgreSQL Schema

```sql
create table if not exists device_command (
  run_id uuid not null,
  command_id uuid not null,
  device_id uuid not null,
  command_type varchar(120) not null,
  request_hash char(64) not null,
  state varchar(32) not null,
  expected_device_version bigint null,
  actual_device_version bigint null,
  target_sim_tick bigint not null,
  correlation_id uuid not null,
  causation_id uuid null,
  result_code varchar(64) null,
  result_payload jsonb null,
  state_version bigint not null default 0,
  created_at timestamptz not null,
  updated_at timestamptz not null,
  primary key (run_id, command_id),
  check (state in ('Accepted','Queued','Executing','Succeeded','Failed','Cancelled','Expired'))
);

create unique index if not exists ux_device_command_hash
  on device_command(run_id, command_id, request_hash);
create index if not exists ix_device_command_pending
  on device_command(run_id, state, target_sim_tick);

create table if not exists occupancy_transfer (
  run_id uuid not null,
  transfer_id uuid not null,
  command_id uuid not null,
  cargo_id uuid not null,
  from_place_id uuid null,
  to_place_id uuid not null,
  state varchar(32) not null,
  expected_occupancy_version bigint not null,
  occupancy_version bigint null,
  state_hash char(64) null,
  created_at timestamptz not null,
  updated_at timestamptz not null,
  primary key (run_id, transfer_id),
  unique (run_id, command_id),
  check (state in ('IntentCreated','Claimed','Prepared','Committed','Rejected','Compensating','Compensated'))
);
create unique index if not exists ux_transfer_active_cargo
  on occupancy_transfer(run_id, cargo_id)
  where state in ('IntentCreated','Claimed','Prepared','Compensating');

create table if not exists device_resource_lock (
  run_id uuid not null,
  lock_id uuid not null,
  resource_type varchar(32) not null,
  resource_id uuid not null,
  owner_type varchar(32) not null,
  owner_id uuid not null,
  state varchar(16) not null,
  lease_until_tick bigint not null,
  state_version bigint not null default 0,
  primary key (run_id, lock_id),
  unique (run_id, resource_type, resource_id),
  check (state in ('Held','Released','Expired'))
);

create table if not exists outbox_message (
  message_id uuid primary key,
  run_id uuid not null,
  aggregate_type varchar(64) not null,
  aggregate_id uuid not null,
  event_type varchar(160) not null,
  event_sequence bigint not null,
  payload jsonb not null,
  state varchar(16) not null default 'Pending',
  attempt_count int not null default 0,
  next_attempt_at timestamptz not null,
  published_at timestamptz null,
  last_error varchar(4000) null,
  unique (run_id, event_sequence),
  check (state in ('Pending','Leased','Published','DeadLetter'))
);
create index if not exists ix_outbox_dispatch
  on outbox_message(state, next_attempt_at);
```

## 4. 事务边界

- T1 Command Admission：`device_command` + `DeviceCommandAccepted` event + `outbox_message`。
- T2 Transfer Commit：`occupancy_transfer` + `world_occupancy` + `CargoTransferCommitted` event + `outbox_message`。
- T3 Recovery Lease：锁租约接管、恢复事件、Outbox 记录必须同事务。
- Dispatcher 不参与 T1/T2；发布失败只更新 Outbox，不回滚业务事实。

## 5. EF Core 规则

- 所有表使用 snake_case 映射；状态字段使用 string converter，禁止数据库 enum 与代码 enum 直接耦合。
- `state_version` 使用 `IsConcurrencyToken()`；更新必须带原版本条件。
- JSONB Payload 仅用于不可查询扩展字段，核心检索字段必须结构化。
- Migration 必须可重复执行，新增索引优先 `CONCURRENTLY`，破坏性变更必须分两阶段发布。

## 6. 可验证门槛

1. `(run_id, command_id)` 重试不新增记录。
2. RequestHash 冲突返回 `DEV-CMD-422`。
3. Active Cargo/Resource 唯一索引阻断并发竞争。
4. Outbox 发布失败不改变 Command/Transfer 状态。
5. Snapshot 恢复后版本单调不回退。
6. 删除 Run 前必须先 Sealed + Archive。
