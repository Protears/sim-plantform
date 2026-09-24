# Dispatch Arbitration Schema

## 1. PostgreSQL Schema

```sql
CREATE TABLE dispatch_arbitration (
    run_id uuid NOT NULL,
    arbitration_id uuid NOT NULL,
    task_id uuid NOT NULL,
    sub_task_id uuid NOT NULL,
    cargo_id uuid NOT NULL,
    route_id uuid NOT NULL,
    priority integer NOT NULL,
    deadline_at timestamptz NOT NULL,
    effective_priority numeric(12,4) NOT NULL,
    status varchar(32) NOT NULL,
    world_version varchar(128) NOT NULL,
    plan_hash varchar(128) NOT NULL,
    conflict_owner_id varchar(128),
    state_version bigint NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL,
    updated_at timestamptz NOT NULL,
    PRIMARY KEY (run_id, arbitration_id),
    UNIQUE (run_id, sub_task_id),
    CHECK (status IN ('Queued','Inspecting','WaitingResource','Granted','Executing','Released','Preempted','Rejected'))
);

CREATE UNIQUE INDEX ux_dispatch_active_cargo
ON dispatch_arbitration(run_id, cargo_id)
WHERE status IN ('Granted','Executing');

CREATE INDEX ix_dispatch_ready
ON dispatch_arbitration(run_id, status, effective_priority DESC, deadline_at, created_at)
WHERE status IN ('Queued','WaitingResource');
```

## 2. 事务边界

- T-DISPATCH：创建 Arbitration + 写事件。
- T-GRANT：校验 WorldVersion、写 Granted、创建 Reservation Outbox。
- T-PREEMPT：写 Preempted、释放 Reservation、重新排队。

## 3. 幂等

`(RunId, ArbitrationId)` 是业务幂等键；同一 SubTask 不允许存在两个活动 Arbitration。

## 4. 保留

已完成/释放记录保留 30 天；Deadlock 相关记录永久保留到 Run 归档后 180 天。
