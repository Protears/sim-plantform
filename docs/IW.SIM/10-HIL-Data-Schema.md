# Part 10：HIL 数据持久化 Schema

## 1. 数据责任

HIL Session 保存会话事实；`hil_signal_frame` 保存收发帧审计；`simulation_event` 保存影响仿真的事件；`external_input_record` 保存可回放输入。Telemetry 不得作为恢复依据。

## 2. PostgreSQL Schema

```sql
create table hil_session (
  session_id uuid primary key,
  simulation_run_id uuid not null,
  state varchar(32) not null,
  state_version bigint not null default 0,
  mapping_hash varchar(128) not null,
  clock_epoch bigint not null default 0,
  last_external_sequence bigint not null default 0,
  connected_at timestamptz,
  disconnected_at timestamptz,
  created_at timestamptz not null default now(),
  constraint uq_hil_session_active unique (simulation_run_id, state)
    deferrable initially immediate
);

create table hil_signal_frame (
  frame_id uuid primary key,
  session_id uuid not null references hil_session(session_id),
  direction varchar(16) not null,
  external_sequence bigint,
  transport_sequence bigint not null,
  clock_epoch bigint not null,
  target_sim_tick bigint,
  mapping_hash varchar(128) not null,
  payload jsonb not null,
  quality varchar(16) not null,
  ack_state varchar(16) not null,
  received_at timestamptz not null,
  unique(session_id, direction, clock_epoch, external_sequence)
);

create index ix_hil_signal_frame_run_tick
  on hil_signal_frame(session_id, target_sim_tick, transport_sequence);
```

## 3. 一致性规则

`state_version` 使用乐观并发；更新必须带 ExpectedVersion。`external_sequence` 唯一约束用于数据库级幂等，内存窗口用于快速拒绝。Session 删除采用软删除或归档，不允许破坏 Run 的事件链。

## 4. 数据保留

运行中数据为 Hot；完成运行后转 Warm；归档包必须包含 MappingHash、ContractVersion、ClockEpoch、输入游标和校验摘要，缺一不可。
