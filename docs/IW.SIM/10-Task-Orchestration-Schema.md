# Task/MCS/WCS 运行态数据 Schema

## 1. 表结构

```sql
create table task_instance (
  run_id uuid not null,
  task_id uuid not null,
  task_type varchar(64) not null,
  cargo_id uuid null,
  source_place_id uuid null,
  target_place_id uuid null,
  priority int not null,
  state varchar(32) not null,
  state_version bigint not null default 0,
  request_hash varchar(128) not null,
  plan_hash varchar(128) null,
  active_sub_task_id uuid null,
  route_id uuid null,
  reservation_id uuid null,
  command_id uuid null,
  world_version bigint not null,
  created_sim_tick bigint not null,
  updated_sim_tick bigint not null,
  primary key (run_id, task_id)
);

create table task_dependency (
  run_id uuid not null,
  task_id uuid not null,
  depends_on_task_id uuid not null,
  primary key (run_id, task_id, depends_on_task_id)
);

create table task_execution_attempt (
  run_id uuid not null,
  task_id uuid not null,
  attempt_no int not null,
  command_id uuid null,
  route_id uuid null,
  reservation_id uuid null,
  result_code varchar(64) null,
  error_code varchar(64) null,
  started_sim_tick bigint null,
  finished_sim_tick bigint null,
  primary key (run_id, task_id, attempt_no)
);

create unique index ux_task_active_cargo
  on task_instance(run_id, cargo_id)
  where cargo_id is not null and state in ('Ready','Reserving','Reserved','Dispatching','Executing','TransferPending');

create index ix_task_ready_priority
  on task_instance(run_id, state, priority desc, created_sim_tick);
```

## 2. 事务边界

- T-MCS：校验依赖、WorldVersion、Capability，写 Task + Dependency。
- T-WCS：写 PlanHash、RouteId、ReservationId、CommandId；不得写 Occupancy。
- T-COMPLETE：仅由 Occupancy Commit 成功后更新 Task，并追加 TaskCompleted Outbox。

## 3. 保留与归档

运行中保留全部 Attempt；Run Sealed 后只读。归档前必须保留最终 PlanHash、RouteHash、StateHash 和错误摘要。
