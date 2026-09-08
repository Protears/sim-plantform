# IW.SIM Part10 - Simulation Data Architecture

> 本章定义 IW.SIM 的数据生命周期、数据边界、事件模型、快照、回放、一致性、PostgreSQL 持久化、冷热数据、数据访问与故障恢复设计。目标不是描述“使用什么数据库”，而是建立可以直接指导 .NET 10 / EF Core 10 工程实现的仿真数据基础设施。

## 1. 文档定位与设计目标

Simulation Data Architecture 是连接 World Model、Simulation Kernel、各类 Runtime、Application Platform、Digital Twin 和 Observability 的数据基础设施层。

仿真系统的数据与普通业务系统有本质区别：数据不仅描述“当前是什么”，还必须能够回答：

- 某一时刻世界处于什么状态；
- 状态为什么发生变化；
- 哪个输入、哪个事件、哪个 Runtime 导致变化；
- 在相同 ProjectVersion、InputSet、Seed 下能否复现；
- 从故障前快照能否恢复；
- 一个测试结果能否追溯到完整运行上下文；
- 大规模运行产生的事件和遥测能否在不影响仿真实时性的情况下长期保存。

因此 IW.SIM 采用：

```text
Model Driven
    +
Event Driven
    +
Time Driven
    +
Snapshot / Replay
    +
Append-only History
```

### 1.1 核心目标

1. **确定性**：相同输入、版本和 Seed 可以得到一致结果。
2. **可追溯**：状态变化能够关联事件、命令、Run、Entity 和 Trace。
3. **可回放**：能够从快照 + 输入事件恢复仿真上下文。
4. **可恢复**：Runtime 故障后可以从一致快照重新启动，而不是依赖数据库当前状态猜测。
5. **可扩展**：运行态热数据、历史事件和分析数据采用不同存储策略。
6. **不阻塞仿真**：持久化不能成为 Simulation Kernel 的同步关键路径瓶颈。

---

## 2. 数据边界

IW.SIM 必须严格区分以下数据：

| 数据域 | 语义 | 权威来源 | 生命周期 |
|---|---|---|---|
| Model Data | 工程模型与配置 | ProjectVersion / SceneVersion | 长期 |
| World State | 当前工业世界事实状态 | World Model | Run 生命周期 |
| Runtime State | 设备、PLC、IO 执行态 | Runtime | Run 生命周期 |
| Input Record | 外部输入及用户刺激 | Input Recorder | Run 生命周期 + 归档 |
| Simulation Event | 仿真内状态变化事实 | Event Store | 长期 |
| Snapshot | 某一一致性点的状态镜像 | Snapshot Store | 长期/策略保留 |
| Result | 测试和实验结果 | Result Store | 长期 |
| Telemetry | 指标、日志、Trace | Observability | 按保留策略 |

禁止以下边界混淆：

```text
ProjectVersion != World State
World State != Runtime State
Runtime State != Event History
Event History != Audit Log
Simulation Event != OpenTelemetry Event
Snapshot != 数据库备份
```

### 2.1 权威性规则

发生冲突时按以下优先级判断：

```text
运行中的 World/Runtime State
        ↓
Simulation Event / Input Record
        ↓
Snapshot
        ↓
持久化 Read Model
```

Snapshot 和查询缓存都不是仿真领域事实的最终权威来源；它们是用于恢复和查询优化的持久化表示。

---

## 3. 数据生命周期

一次 SimulationRun 的数据生命周期统一为：

```text
Create Run
   |
Resolve ProjectVersion
   |
Create Input Set
   |
Initialize Runtime
   |
Run
   |
+--> Capture Event
+--> Update State
+--> Record External Input
+--> Capture Metrics
+--> Periodic Snapshot
   |
Stop / Complete / Fail
   |
Finalize Result
   |
Seal Run Data
   |
Archive / Retain / Purge
```

Run 完成后必须进入 **Sealed** 数据阶段。Sealed Run 的历史事件、输入、快照和结果禁止被业务 API 原地修改。

---

## 4. Simulation Data Envelope

所有持久化仿真事件、外部输入和关键状态变化采用统一 Envelope，避免不同 Runtime 各自定义不可关联的数据格式。

```csharp
public sealed record SimulationDataEnvelope(
    Guid EventId,
    Guid RunId,
    long Sequence,
    long SimTick,
    double SimTime,
    string EventType,
    string SourceType,
    string SourceId,
    string? CorrelationId,
    string? CausationId,
    int SchemaVersion,
    DateTimeOffset RecordedAt,
    string PayloadType,
    ReadOnlyMemory<byte> Payload);
```

### 4.1 字段约束

| 字段 | 约束 |
|---|---|
| EventId | 全局唯一，不能复用 |
| RunId | 必填，所有运行数据必须归属 Run |
| Sequence | Run 内单调递增 |
| SimTick | 由 Simulation Clock 产生 |
| SimTime | 与 SimTick 对应，不使用墙上时钟替代 |
| EventType | 稳定事件类型标识 |
| SourceId | 产生事件的 Entity/Runtime/Integration |
| CorrelationId | 同一业务/测试链路关联 |
| CausationId | 指向直接原因事件或命令 |
| SchemaVersion | Payload 演进版本 |
| RecordedAt | 仅用于基础设施审计，不参与仿真排序 |

`RecordedAt` 不能参与确定性计算；任何仿真逻辑必须使用 `SimTick/SimTime`。

---

## 5. Event Store 设计

Event Store 采用 Append-only 原则。已写入事件禁止 Update/Delete，纠错通过新事件表达。

### 5.1 事件排序

Run 内事件的唯一排序键定义为：

```text
(SimTick, Priority, Sequence)
```

其中：

- `SimTick`：仿真时间；
- `Priority`：同一 Tick 内的确定性处理优先级；
- `Sequence`：最终稳定序号。

数据库写入顺序不能反过来决定仿真顺序。

### 5.2 事件类型分类

```text
Domain Event
    CargoCreated
    CargoMoved
    OccupancyChanged

Runtime Event
    DeviceStarted
    MotionCompleted
    RuntimeFaulted

Control Event
    PlcSignalChanged
    CommandAccepted
    CommandRejected

Integration Event
    WcsMessageReceived
    ExternalConnectionChanged

System Event
    RunStarted
    SnapshotCreated
    RunCompleted
    RunFailed
```

### 5.3 事件不可变与版本化

事件类型一旦进入已发布版本，不允许改变既有字段语义。Schema 演进采用：

```text
EventType + SchemaVersion
```

消费者必须显式声明支持的 SchemaVersion；迁移工具负责旧事件向新 Read Model 的兼容，而不是修改历史事件。

---

## 6. External Input Record

外部系统输入必须与普通 Simulation Event 区分，因为它是确定性重放的输入边界。

```csharp
public sealed record SimulationInputRecord(
    Guid InputId,
    Guid RunId,
    long Sequence,
    long SimTick,
    string Source,
    string InputType,
    string PayloadHash,
    byte[] Payload,
    string SchemaVersion);
```

典型输入：

- WCS 指令；
- PLC 外部输入；
- HIL 设备输入；
- 用户测试刺激；
- 外部设备反馈；
- 定时器/随机数输入的记录结果。

### 6.1 Replay 原则

Replay 不重新连接真实外部系统，而采用：

```text
Original External Input
        |
Input Recorder
        |
Input Set
        |
Replay Adapter
        |
Simulation Runtime
```

因此“复现”与“重新联调”是两个不同运行模式。

---

## 7. World State 持久化模型

World State 由 World Model 管理，Data Layer 只负责其持久化和恢复。

建议采用：

```text
world_entity
world_relation
world_property
world_location
world_occupancy
```

核心约束：

- `world_entity` 保存稳定身份和类型；
- 位置、占用、关系必须具有明确版本或 Run 归属；
- 不把完整 Runtime 对象序列化成不可查询的大 JSON 作为唯一模型；
- JSONB 只用于扩展属性，核心关联字段使用结构化列。

对于运行态数据，推荐使用：

```text
run_world_state
run_runtime_state
```

而不是直接覆盖工程模型表。

---

## 8. Runtime State 持久化

Runtime State 是可恢复状态，但不是完整事件历史。

```text
DeviceRuntimeState
{
    RunId,
    EntityId,
    Version,
    Mode,
    CurrentTaskId,
    MotionState,
    AlarmState,
    IoState,
    UpdatedSimTick
}
```

### 8.1 State Version

每个可并发修改的运行态聚合必须维护 `Version`：

```text
Read Version = 17
       |
Command ExpectedVersion = 17
       |
Update -> Version 18
```

如果 ExpectedVersion 不匹配，返回并发冲突，不允许旧客户端覆盖新状态。

---

## 9. Snapshot 一致性模型

Snapshot 不是简单地“把当前对象序列化一次”。它必须代表一个明确的一致性点。

Snapshot 必须包含：

```text
Snapshot
 ├── RunId
 ├── SnapshotId
 ├── BaseEventSequence
 ├── SimTick
 ├── SimTime
 ├── WorldState
 ├── RuntimeState
 ├── SchedulerState
 ├── RandomState
 ├── InputCursor
 ├── SchemaVersion
 └── Checksum
```

### 9.1 Snapshot 原子性

Snapshot 创建采用两阶段策略：

```text
1. Freeze Simulation at Tick T
2. Capture World + Runtime + Scheduler
3. Capture Event Sequence S
4. Persist Snapshot Manifest
5. Verify Checksum
6. Mark Snapshot Committed
7. Resume Simulation
```

关键规则：

```text
BaseEventSequence = S
SimTick = T
```

恢复时只能从 Snapshot 的一致性点开始继续消费 `S+1` 之后的事件/输入。

### 9.2 Snapshot 失败

Snapshot 写入失败不能影响已经完成的仿真 Tick。失败必须：

- 记录 SnapshotFailed；
- 保留上一有效 Snapshot；
- 标记当前 Run 的恢复能力降级；
- 不允许生成“看起来成功但内容不完整”的 Snapshot。

---

## 10. Restore / Recovery

恢复分为三种语义：

| 模式 | 目的 | 外部输入 |
|---|---|---|
| Snapshot Restore | 从一致状态继续运行 | Replay/隔离输入 |
| Full Replay | 从起点重演 | 历史 Input Set |
| Fresh Run | 全新实验 | 新输入 |

恢复流程：

```text
Load Snapshot
    |
Verify Checksum / Schema
    |
Restore World
    |
Restore Runtime
    |
Restore Scheduler
    |
Restore Random State
    |
Restore Input Cursor
    |
Replay Input/Event S+1...
    |
Resume
```

禁止只恢复 World State 而忽略 Scheduler、Random State 或 Input Cursor，否则可能出现“视觉状态相同但后续行为不同”。

---

## 11. PostgreSQL 持久化设计

生产环境采用 PostgreSQL，推荐逻辑分区：

```text
project_*          -- 工程配置
run_*              -- 运行元数据
simulation_event   -- 事件
simulation_input   -- 外部输入
simulation_snapshot -- 快照
simulation_result  -- 结果
```

### 11.1 Event 表建议

```sql
simulation_event
-----------------------------
id uuid primary key
run_id uuid not null
sequence bigint not null
sim_tick bigint not null
sim_time double precision not null
event_type varchar(160) not null
source_type varchar(80) not null
source_id varchar(160) not null
correlation_id uuid null
causation_id uuid null
schema_version integer not null
payload_type varchar(160) not null
payload jsonb not null
recorded_at timestamptz not null
```

约束与索引：

```text
UNIQUE(run_id, sequence)
INDEX(run_id, sim_tick, sequence)
INDEX(run_id, event_type, sim_tick)
INDEX(correlation_id)
```

大型 Run 的 Event Store 应按 `run_id` 或时间范围进行分区评估，不能默认一个无限增长的大表。

### 11.2 Snapshot 表建议

```sql
simulation_snapshot
-----------------------------
id uuid primary key
run_id uuid not null
base_event_sequence bigint not null
sim_tick bigint not null
sim_time double precision not null
schema_version integer not null
storage_uri text not null
payload_hash varchar(128) not null
status varchar(32) not null
created_at timestamptz not null
```

`status` 至少支持：

```text
Writing -> Verifying -> Committed
                    \-> Failed
```

---

## 12. Transaction Boundary

IW.SIM 不允许使用一个巨型数据库事务包裹整个 Simulation Tick，也不允许让数据库提交成功与仿真状态提交成功之间产生无法识别的不一致。

采用：

```text
Simulation Transaction
       |
       +-- State Mutation
       +-- Event Generation
       +-- Input Cursor Update
       |
       v
Persistence Boundary
```

### 12.1 推荐提交策略

对于必须持久化的关键事件：

```text
Runtime State Mutation
        |
Event Buffer
        |
Persistence Writer
        |
PostgreSQL Transaction
```

一个 Persistence Batch 可以包含同一 Run 的多个事件，但必须保证：

```text
Sequence 连续
Event Payload 完整
RunId 一致
```

如果数据库提交失败，Runtime 不能静默丢弃事件；必须进入明确的 `PersistenceDegraded` 状态，并按照策略暂停、缓冲或终止 Run。

---

## 13. Outbox / Inbox 与外部一致性

Simulation Event 写入数据库后，需要通知 Application、Result、Digital Twin 或其他消费者时，不能依赖“先写数据库再直接发消息”的非原子流程。

推荐：

```text
Simulation Event
      |
PostgreSQL Transaction
  +---+----------------+
  |                    |
Event Store          Outbox
                       |
                 Dispatcher
                       |
             Application / Twin / Result
```

对于外部输入采用 Inbox：

```text
External Message
      |
Inbox Deduplication
      |
Input Record
      |
Runtime
```

幂等键建议由：

```text
Source + ExternalMessageId
```

或协议能够提供稳定序号时使用：

```text
Source + SessionId + Sequence
```

防止 WCS/PLC 重发造成重复刺激。

---

## 14. Hot / Warm / Cold 数据策略

不同数据不能采用同一保留策略。

```text
HOT
  Runtime State / Active Run / Recent Events

WARM
  Completed Run / Snapshot / Test Result

COLD
  Long-term Event / Experiment Dataset / Artifact
```

建议：

| 数据 | HOT | WARM | COLD |
|---|---|---|---|
| Runtime State | Memory/Redis | - | - |
| Event | PostgreSQL | PostgreSQL/归档 | Parquet/Object Storage |
| Snapshot | Local/Object | Object | Object |
| Result | PostgreSQL | PostgreSQL | Parquet |
| Trace | OTLP Backend | 压缩归档 | 按策略删除 |

Redis 只能作为运行态缓存，不作为唯一事实来源。

---

## 15. Data Access API

Data Layer 对上提供按职责划分的接口，而不是暴露 DbContext：

```csharp
public interface ISimulationEventStore
{
    ValueTask AppendAsync(
        IReadOnlyList<SimulationDataEnvelope> events,
        CancellationToken cancellationToken);

    IAsyncEnumerable<SimulationDataEnvelope> ReadAsync(
        Guid runId,
        long fromSequence,
        CancellationToken cancellationToken);
}

public interface ISimulationSnapshotStore
{
    Task<SnapshotDescriptor> CreateAsync(
        SnapshotRequest request,
        CancellationToken cancellationToken);

    Task RestoreAsync(
        Guid snapshotId,
        CancellationToken cancellationToken);
}
```

Application Platform 通过 Data Service 获取 Read Model；Simulation Kernel/Runtime 通过明确的运行时接口写入 Event/State，不直接操作 EF Core Entity。

---

## 16. Query Read Model

高频页面查询不能实时重放整个 Event Store。

推荐维护：

```text
Event Store
    |
Projection Worker
    |
Read Model
```

例如：

```text
run_device_summary
run_cargo_summary
run_alarm_summary
run_task_summary
run_performance_summary
```

Read Model 可以异步最终一致，但页面必须显示：

```text
lastProcessedSequence
```

这样用户可以知道当前查询结果是否已经追上仿真事件。

对于需要严格实时的数据，直接从 Runtime Query Facade 获取，而不是等待数据库 Projection。

---

## 17. Result 数据模型

SimulationRun 与 Result 解耦：

```text
SimulationRun
     |
     +-- RunResult
          +-- Metric
          +-- AssertionResult
          +-- ArtifactReference
```

RunResult 至少记录：

```text
RunId
ProjectVersionId
SceneVersionId
RunProfileId
InputSetId
Seed
StartSimTick
EndSimTick
Status
FailureReason
CreatedAt
```

结果必须能够反向定位到运行上下文，不允许只保存一个“最终 KPI 数字”而丢失计算来源。

---

## 18. Deterministic Replay 校验

确定性测试不能只比较最终状态，还应比较关键事件序列。

推荐校验：

```text
Run A
  Event Sequence Hash
        |
        +---- compare ----+
                         Run B
                           Event Sequence Hash
```

分层校验：

1. 输入序列一致；
2. Event Sequence 一致；
3. 关键 World State Hash 一致；
4. 最终 Result 一致。

如果输入一致但 Event Sequence 不一致，测试必须判定为 Determinism Failure，即使最终 KPI 恰好相同。

---

## 19. Hash 与校验

为防止事件、快照和实验数据被静默修改，建议保存：

```text
PayloadHash
SnapshotHash
RunManifestHash
EventSequenceHash
```

Run 完成时生成 Manifest：

```text
RunManifest
 ├── ProjectVersionHash
 ├── SceneVersionHash
 ├── RunProfileHash
 ├── InputSetHash
 ├── EventSequenceHash
 ├── SnapshotHashes
 └── ResultHash
```

Manifest 是复现和归档的完整性基线。

---

## 20. 并发控制

数据层并发控制必须区分三类并发：

### 20.1 Application 并发

两个用户同时修改 ProjectVersion 时使用乐观锁：

```text
Version 12
  |
User A -> 13
User B -> reject ExpectedVersion=12
```

### 20.2 Runtime Command 并发

同一个 Run 的 Start/Stop/Pause/Reset 使用 Run 级 Command Lock；不能依赖数据库行锁覆盖整个运行周期。

### 20.3 Event Writer 并发

同一个 Run 原则上只有一个逻辑 Sequence Writer。多个生产者通过 Runtime Event Buffer 汇聚，由单一排序/持久化出口分配 Sequence。

这样避免：

```text
Producer A -> Sequence 101
Producer B -> Sequence 103
Producer C -> Sequence 102
```

导致历史顺序与仿真顺序不一致。

---

## 21. Backpressure 与数据丢失策略

事件写入速度低于仿真产生速度时，必须显式处理背压。

```text
Runtime
   |
Bounded Event Buffer
   |
Persistence Writer
   |
PostgreSQL
```

策略分级：

| 数据 | 背压策略 |
|---|---|
| Domain Event | 不允许丢失 |
| External Input | 不允许丢失 |
| Snapshot | 可以延迟 |
| Debug Log | 可以采样/丢弃 |
| UI Telemetry | 可以降采样 |

当不可丢失事件达到 Buffer 上限：

```text
Normal
  -> PersistenceDegraded
  -> Throttled / Paused
  -> Failed
```

不能通过无限扩大内存队列解决问题，否则长时间运行最终会造成 OOM。

---

## 22. 数据保留与清理

Retention 必须以 Run、ProjectVersion 和测试结果为边界，而不是随机删除单行数据。

示例策略：

```text
Active Run
    -> 保留全部关键事件
Completed Run
    -> 保留 Event + Result + Snapshot
Archived Run
    -> Event 压缩/对象存储
Expired Run
    -> 按 Project/租户策略清理
```

删除必须经过：

```text
Retention Policy
   |
Eligibility Check
   |
Archive (optional)
   |
Delete
   |
Audit Record
```

不得删除仍被 ProjectVersion、TestResult、Artifact 或其他审计记录引用的数据。

---

## 23. 故障场景

### 23.1 PostgreSQL 暂时不可用

```text
Detect
  |
PersistenceDegraded
  |
Buffer / Pause
  |
Retry with Backoff
  |
Recovered -> Flush
  |
Resume
```

超过最大缓冲能力后必须 Fail Run，并保留故障前最后一个有效 Snapshot。

### 23.2 Snapshot Storage 不可用

不影响已经完成的 Simulation Tick；Snapshot 状态为 Failed，Run 标记恢复能力降级。后续仍可继续运行，但必须明确告警。

### 23.3 Event 写入成功、Outbox Dispatcher 失败

Event Store 保持成功状态，Outbox 保留待发送记录；Dispatcher 恢复后继续投递。消费者必须幂等。

### 23.4 Runtime 已更新、数据库事务失败

不能伪装成成功。Runtime 必须进入 PersistenceDegraded，并依据 RunProfile 的一致性策略暂停或终止。若运行允许纯内存模式，必须把该模式显式标记为“不可持久化/不可保证恢复”。

---

## 24. 与 Simulation Kernel 的契约

Simulation Kernel 只关心以下数据接口：

```text
SimClock
EventScheduler
Runtime State Mutation
Event Emit
Input Consume
Snapshot Barrier
```

Kernel 不依赖：

```text
PostgreSQL
EF Core
Redis
Object Storage
```

因此可以在单元测试中使用 InMemory Event Store，也可以在生产环境使用 PostgreSQL，而不改变仿真核心语义。

---

## 25. 与 Application Platform 的契约

Application Platform 可以：

```text
Create Run
Query Run
Query Events
Query History
Create Snapshot
Restore Snapshot
Get Result
```

但不能：

```text
直接修改 Event Store
直接修改 Runtime State
绕过 Run Command 修改运行状态
```

Application 的查询通过 Read Model；控制通过 Command；恢复通过 Snapshot Service。

---

## 26. 与 Digital Twin 的契约

Digital Twin Integration 不应直接订阅数据库表变化，而应订阅稳定的数据事件/Projection：

```text
Simulation Event
       |
   Twin Adapter
       |
External Twin
```

Twin 同步失败不能回滚仿真本身。采用：

```text
At-least-once Delivery
+
Idempotent Consumer
+
LastProcessedSequence
```

确保外部 Twin 暂时离线后可以从最后确认序号继续同步。

---

## 27. 与 Observability 的关系

Simulation Data 与 OpenTelemetry 是两套不同目的的数据体系：

| Simulation Data | OpenTelemetry |
|---|---|
| 仿真事实 | 系统运行观测 |
| 需要确定性 | 允许采样 |
| 长期回放 | 按保留策略 |
| Event Sequence | Trace/Span |
| SimTime | Wall Clock |

两者通过统一关联字段连接：

```text
RunId
EntityId
CorrelationId
EventId
TraceId
```

不能使用 Trace/Log 代替 Simulation Event Store。

---

## 28. .NET 10 / EF Core 10 工程映射

推荐项目边界：

```text
IW.SIM.Data.Abstractions
IW.SIM.Data.PostgreSql
IW.SIM.Data.Serialization
IW.SIM.Data.Projection
IW.SIM.Data.Tests
```

### 28.1 EF Core 原则

- EF Core 用于模型、Run 元数据、Result、Projection 等事务型数据；
- 高频 Event Append 可以使用批量写入/专用 SQL，不能强制每个事件创建一个 DbContext 事务；
- Migration 管理结构变更；
- Event Payload 使用 JSONB，但核心索引字段结构化；
- 查询采用 no-tracking Read Model；
- 大型历史查询必须分页，并限制最大时间范围/结果数。

### 28.2 序列化

事件 Payload 必须携带稳定的 `PayloadType + SchemaVersion`。序列化器升级不能改变旧事件的解释。

---

## 29. 数据访问性能基线

数据层至少建立以下可观测指标：

```text
simulation_event_append_latency
simulation_event_buffer_depth
simulation_event_persist_failure_total
snapshot_create_duration
snapshot_restore_duration
projection_lag_sequence
run_data_size_bytes
replay_events_per_second
```

性能评估必须区分：

```text
Runtime critical path
Persistence path
Query path
Archive path
```

任何优化不能通过牺牲 Runtime 确定性换取数据库吞吐。

---

## 30. 测试与验收矩阵

| 场景 | 验证内容 | 通过标准 |
|---|---|---|
| Event Append | Sequence/唯一性 | 无重复、无断序 |
| Duplicate Input | Inbox 幂等 | 同一输入只产生一次有效刺激 |
| Snapshot | 一致性 | 恢复后关键状态一致 |
| Replay | 确定性 | 输入/Event/State Hash 符合预期 |
| DB Failure | 背压与恢复 | 不静默丢关键事件 |
| Projection Lag | 最终一致 | Lag 可观测且可恢复 |
| Concurrent Command | 乐观锁 | 旧版本被拒绝 |
| Schema Upgrade | 事件兼容 | 历史事件仍可读取 |
| Retention | 引用完整性 | 无悬挂引用 |
| Large Run | 分区/分页 | 查询不扫描无限历史 |

### 30.1 最低自动化测试集

必须至少覆盖：

1. 同一 InputSet + Seed 执行两次，EventSequenceHash 相同；
2. Snapshot 在 Tick T 创建后恢复，继续执行结果与完整 Replay 一致；
3. 重复外部消息不会生成重复 InputRecord；
4. PostgreSQL 短暂不可用时关键事件不丢失；
5. Event SchemaVersion 升级后旧数据仍可查询；
6. 两个并发 Command 对同一 Run 不会产生非法状态转换；
7. Projection 落后时 Application 能报告 `lastProcessedSequence`；
8. Run Sealed 后历史 Event/Result 不可通过普通 API 修改。

---

## 31. 关键 ADR

### ADR-DATA-001：Simulation Event Append-only

**Decision:** 仿真事件不可修改、不可物理删除。

**Reason:** 保证追溯、Replay、确定性校验和故障分析。

### ADR-DATA-002：Snapshot 是一致性恢复点

**Decision:** Snapshot 必须绑定 `BaseEventSequence + SimTick`，并包含 Scheduler、Random State 和 Input Cursor。

**Reason:** 仅恢复世界对象无法保证后续行为一致。

### ADR-DATA-003：Runtime 与数据库解耦

**Decision:** Simulation Kernel 不依赖 PostgreSQL/EF Core。

**Reason:** 保证实时执行、测试隔离和未来不同 Hosting 模式的可替换性。

### ADR-DATA-004：外部输入单独记录

**Decision:** WCS/PLC/HIL/用户刺激统一进入 Input Record。

**Reason:** 外部输入是 Deterministic Replay 的边界。

### ADR-DATA-005：Event 与 Telemetry 分离

**Decision:** OpenTelemetry 不承担 Simulation Event Store 职责。

**Reason:** Telemetry 可采样，而仿真事实不能丢失。

---

## 32. 本章设计闭环

```text
ProjectVersion
      |
      v
SimulationRun
      |
      +---- Input Set --------+
      |                       |
      v                       v
Simulation Kernel -----> Event Buffer
      |                       |
      |                       v
      |                 Event Store
      |                       |
      v                       +----> Projection ----> Query
World / Runtime              |
      |                       +----> Outbox ----> Application / Twin
      |
      +---- Snapshot Barrier
                 |
                 v
          Snapshot Store
                 |
                 v
          Restore / Replay
                 |
                 v
          Simulation Kernel
```

该闭环明确了：

- 谁产生数据：Kernel / Runtime / Integration；
- 谁决定仿真顺序：Simulation Kernel；
- 谁保存事实：Event Store；
- 谁保存恢复点：Snapshot Store；
- 谁提供查询：Projection / Data Service；
- 谁负责外部输入重演：Input Recorder / Replay Adapter；
- 谁负责外部投递：Outbox Dispatcher；
- 谁负责可观测性：OpenTelemetry；
- 谁负责工程操作：Application Platform。

---

## 33. 验收标准

Part10 达到可评审状态必须满足：

- [x] World State / Runtime State / Event / Input / Snapshot / Result 边界明确；
- [x] Event Envelope、Sequence、Causation、Correlation、SchemaVersion 明确；
- [x] Append-only Event Store 规则明确；
- [x] Snapshot 一致性点及恢复语义明确；
- [x] PostgreSQL 表、约束、索引和分区策略有实现依据；
- [x] Transaction / Outbox / Inbox 一致性边界明确；
- [x] Hot/Warm/Cold 数据策略明确；
- [x] Backpressure 和关键数据丢失策略明确；
- [x] Replay / Determinism 校验明确；
- [x] Application / Kernel / Digital Twin / Observability 边界明确；
- [x] .NET 10 / EF Core 10 工程映射明确；
- [x] 自动化测试和验收矩阵明确。

> 本章不以“数据库能存下来”为验收目标，而以“仿真能够确定执行、完整追溯、可靠恢复、可回放、可分析，并且数据层不会反向侵入 Simulation Kernel”为验收目标。
