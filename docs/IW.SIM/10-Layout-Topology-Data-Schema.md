# Layout 拓扑数据 Schema

## PostgreSQL Schema

```sql
create table layout_snapshot (
    layout_snapshot_id uuid primary key,
    layout_id uuid not null,
    version integer not null,
    layout_hash varchar(64) not null,
    status varchar(16) not null,
    payload jsonb not null,
    created_at timestamptz not null,
    unique (layout_id, version),
    unique (layout_snapshot_id, layout_hash)
);

create table layout_link (
    layout_snapshot_id uuid not null references layout_snapshot(layout_snapshot_id),
    link_id uuid not null,
    from_type varchar(16) not null,
    from_id uuid not null,
    to_type varchar(16) not null,
    to_id uuid not null,
    direction varchar(16) not null,
    cost numeric(18,6) not null default 0,
    primary key (layout_snapshot_id, link_id)
);

create index ix_layout_link_from on layout_link(layout_snapshot_id, from_type, from_id);
create index ix_layout_link_to on layout_link(layout_snapshot_id, to_type, to_id);
```

## 事务边界

- Layout Validate + Snapshot Persist：T-Layout-1。
- Device Bind + BindingSnapshot Persist：T-Layout-2。
- Route Reservation + ResourceLock：T-Traffic-1。
- 不允许 Runtime 在事务外写入 Layout 事实。
