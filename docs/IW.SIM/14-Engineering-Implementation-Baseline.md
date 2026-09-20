# Part 14：工程实施基线与模块装配规则

## 1. 目标与适用范围

本文件把 Part02/03/05/06/07/08/10/11/12 的设计契约收敛为 .NET 10 模块装配基线，作为编码、代码评审、测试和部署的共同入口。它规定程序集边界、线程模型、端口适配、事务边界、正式数据模型、运行时装配和最小可交付切片。

本轮新增实施约束：

- Kernel 是唯一 `SimTick` 权威；
- 所有异步输入必须经有界入口和 TickBoundary；
- Snapshot 只能在完整事实提交边界创建；
- PLC—设备反馈必须可沿 `CycleSequence → CommandId → TransferId → OccupancyVersion → ProcessImageVersion` 追踪；
- Command、Transfer、Lock、Event、Outbox 使用 Part10 正式数据 Schema v2；
- Recovery 必须遵循事实校验、Cursor、World/Device、PLC Image、Scheduler、Replay 顺序；
- Architecture/Contract/Migration/Replay/稳定性测试是发布前强制门禁；
- Run Event Query、SignalR 补发和 Replay 统一使用 EventSequence 游标；
- RunProfile 在运行前必须完成 SchemaVersion、MappingHash、Capability、ProtocolProfile 校验并冻结；
- Run 运维操作统一经 `IRunLifecycleCoordinator`，使用 `OperationId + ExpectedStateVersion` 保证幂等和并发一致性；
- Operational Audit 是运维动作权威，Telemetry 不得用于推断业务事实；
- Trace 必须能够沿 `CycleSequence → CommandId → TransferId → OccupancyVersion → ProcessImageVersion → EventSequence` 反查完整链路；
- 发布前必须提供配置摘要、Schema 版本、Recovery/Replay Hash、Trace 抽样和审计摘要组成的证据包；
- Device Runtime、PLC Mapper、Signal IO、Recovery、Replay 必须读取同一个不可变 `DeviceBindingSnapshot`；
- Definition、Configuration、Capability、PointMapping、ControllerProfile 的规范化摘要必须生成 `DeviceConfigHash/MappingHash`，运行中不得热更新确定性字段；
- 设备实例必须经过 `Defined → Bound → Initializing → Ready` 才能接收普通命令；
- 设备模型装配和点位映射测试是进入 PLC/HIL 联调的前置门禁。

配套实施文档：

- `03-Kernel-Command-Ordering-and-Determinism.md`
- `03-Kernel-Input-Backpressure-and-Recovery.md`
- `03-Kernel-Run-Recovery-Protocol-v2.md`
- `03-Run-Lifecycle-and-Operational-Command-State-Machine.md`
- `05-Device-Definition-and-Instance-Binding.md`
- `05-Device-Configuration-Schema.md`
- `05-Device-Instance-Lifecycle-and-Health.md`
- `06-Device-Capability-and-Point-Mapping-Contract.md`
- `10-Kernel-Snapshot-Consistency-Contract.md`
- `10-Formal-Operational-Schema-v2.md`
- `10-Run-Audit-and-Operational-Trace-Schema.md`
- `06-PLC-Device-Command-Feedback-Trace.md`
- `06-PLC-Feedback-Contract-Test-Matrix-v2.md`
- `14-Project-Structure-and-Dependency-Verification.md`
- `14-Architecture-Contract-Test-Implementation-v2.md`
- `14-Engineering-Task-Slice-D.md`
- `14-Engineering-Slice-Acceptance-Matrix.md`
- `14-Device-Model-Commissioning-Test-Matrix.md`
- `14-Run-Profile-and-Configuration-Schema.md`
- `14-Operational-Readiness-and-Release-Gates.md`
- `11-Run-Event-Query-and-Replay-Contract-v2.md`
- `11-Trace-Query-and-Operations-Api-Contract.md`
- `ADR/ADR-016-Commit-Then-Publish-Facts.md`
- `ADR/ADR-017-Kernel-Is-Only-SimTick-Authority.md`
- `ADR/ADR-018-Formal-Data-and-Test-Gates.md`
- `ADR/ADR-019-Recovery-And-Event-Cursor-Authority.md`
- `ADR/ADR-020-Run-Configuration-and-Operational-Trace-Authority.md`
- `ADR/ADR-021-Device-Binding-Snapshot-Authority.md`

## 2. 解决方案与程序集边界

```text
src/
  Logistics.Simulation.Domain/
  Logistics.Simulation.Contracts/
  Logistics.Simulation.Application/
  Logistics.Simulation.SimulationKernel/
  Logistics.Simulation.DeviceRuntime/
  Logistics.Simulation.PlcRuntime/
  Logistics.Simulation.SignalIo/
  Logistics.Simulation.Communication/
  Logistics.Simulation.Hil/
  Logistics.Simulation.Data/
  Logistics.Simulation.Api/
  Logistics.Simulation.Worker/
tests/
  *.UnitTests/
  *.IntegrationTests/
  *.ContractTests/
  *.ArchitectureTests/
  *.DataMigrationTests/
  *.ReplayDeterminismTests/
```

依赖规则：

- Domain 只依赖 BCL；禁止依赖 EF Core、ASP.NET、SignalR、PLC SDK。
- SimulationKernel 依赖 Domain，不依赖 Api、Data；通过端口读取输入和写入事件。
- DeviceRuntime、PlcRuntime、SignalIo 依赖 Domain 与明确的 Application Ports，不互相引用实现程序集。
- Communication/Hil 只能通过 Ports 向 Application/SignalIo 提供帧和信号；禁止直接写 World State。
- Data 实现 Application/Domain 定义的 Store 接口；EF Core 只出现在 Data。
- Api/Worker 负责组合根和生命周期，不在控制器中执行设备算法。
- DeviceRuntime 不得读取可变配置源；只能读取 `DeviceBindingSnapshot`。

## 3. 组合根与运行时装配

```csharp
public interface ISimulationRuntimeFactory
{
    SimulationRuntime Create(SimulationRuntimeOptions options);
}

public sealed record SimulationRuntimeOptions(
    Guid RunId,
    string ProfileName,
    int MaxDevices,
    TimeSpan TickBudget,
    bool EnableHil);
```

组合根必须完成：

1. 校验 RunProfile、MappingHash、ProtocolProfile。
2. 建立单一 `SimulationRuntime` 实例和单一 Kernel 主循环。
3. 注册 Device Runtime、PLC Runtime、Signal/IO、Communication、HIL 端口。
4. 绑定 EventStore、SnapshotStore、InputRecordStore、Outbox。
5. 启动健康检查和故障升级策略。
6. 注册 `IInputIngress`、`ISimulationCommandQueue` 和 Snapshot Coordinator，确保所有异步输入与恢复路径经过统一边界。
7. 注册正式数据迁移检查器，确认 SchemaVersion 与当前运行时兼容。
8. 注册 `IKernelRecoveryCoordinator` 和 `IRunEventReplayService`，禁止外部模块旁路恢复或直接分页事件。
9. 注册 `IRunLifecycleCoordinator`、`IOperationalAuditStore` 和 Trace Query 组件，确保运维命令、审计和业务链路使用统一入口。
10. 将 Profile 冻结摘要、Schema 版本、MappingHash 和 Capability 清单写入 Run 初始化事实。
11. 解析 Definition/Configuration/Capability/PointMapping/ControllerProfile 并生成 `DeviceBindingSnapshot`。
12. 运行前执行设备模型装配、点位映射、健康初始状态和 Snapshot Hash 校验。

## 4. 线程与调度模型

- Kernel 主循环是唯一推进 SimTick 的线程。
- PLC 逻辑、设备状态演化和 Occupancy Commit 必须在 Kernel 调度上下文内完成；异步 I/O 只能投递输入事件，不得直接改变领域状态。
- Communication/HIL 使用独立 I/O 线程或 async socket，但通过有界 Channel 将数据交给 InputIngress。
- Snapshot/Outbox/Audit 持久化为后台消费者；持久化延迟不能阻塞 Kernel，但必须提供 `PersistenceLag` 和降级门槛。
- 同一 Run 的业务事件按 `(SimTick, Phase, Priority, SourceId, LocalSequence)` 排序；不得依赖线程完成顺序。

## 5. 一致性与恢复基线

- 输入必须经过 `InputCommit` 才能影响当前 Tick。
- Snapshot 只能在 `OccupancyCommit` 和 `EventAppend` 完成后创建。
- 恢复顺序为：事实校验 → Cursor → World/Device → PLC Image → Scheduler Queue → Replay。
- 新 Epoch 建立后，旧 Epoch 输入全部拒收。
- Outbox 发布失败不能回滚已提交事实，也不能重新执行设备动作。
- Run 事件查询和 SignalR 补发都以 Event Sequence 为游标，不以时间戳分页。
- Recovery/Replaying 期间不得发布 Completed、CargoTransferCommitted 等对外完成事件。
- RunProfile 在 Recovery/Replay 中视为 Frozen，只能读取，不能热更新确定性字段。
- Device Runtime 恢复必须先校验 `DeviceBindingSnapshotId/MappingHash`，不匹配时进入 Failed，不得尝试使用当前最新配置继续恢复。

## 6. 数据迁移与兼容基线

- Migration 必须遵循 Expand → Backfill → Switch → Contract 四阶段。
- 新增字段先允许 NULL 或默认值，再在数据回填完成后提升约束。
- 状态枚举扩展必须先兼容读取，再切换写入，避免旧 Worker 无法读取新状态。
- 破坏性索引和大表变更必须提供回滚方案及窗口评估。
- 运行时启动前执行 SchemaVersion 校验；不兼容时拒绝进入 Running。
- Operational Audit 仅追加写；Audit/Trace 表不得被业务清理任务直接物理删除。
- DeviceBindingSnapshot、DeviceConfigHash、MappingHash 必须写入 Run 初始化事实；缺失时禁止进入 Ready。

## 7. 最小工程切片

### Slice A：纯仿真

Kernel + SignalIo + DeviceRuntime + WorldModel + EventStore。验收 PLC/HIL 关闭时，普通输送机完成货载转移并生成可重放事件。

### Slice B：PLC 联调

增加 PlcRuntime + ProcessImage + S7/Modbus Adapter。验收 PLC 输出驱动设备，设备传感器反馈回 PLC，周期 Hash 稳定，并能沿 Trace 查询完整链路。

### Slice C：HIL

增加 Communication + HilSession + TimeNormalizer + Ack/SafetyGuard。验收断线、迟到输入、旧 Epoch、Critical Output Ack 超时。

### Slice D：Kernel 工程闭环

增加确定性排序、背压恢复、Snapshot 一致性和 PLC—设备 Trace。验收不同输入到达顺序下 Replay Hash 一致，并能在故障后恢复至可验证状态。

### Slice E：数据与交付闭环

增加正式 PostgreSQL Schema v2、EF Core Migration、Architecture/Contract/Migration/Replay 测试、Run Event Query 和 SignalR 补发。验收命令、Transfer、Outbox 的事务一致性及断线补发。

### Slice F：运行治理闭环

增加 RunProfile Schema、Run 生命周期运维命令、Operational Audit、Trace Query 和 Operational Readiness 门禁。验收配置冻结、运维命令幂等、审计完整、Trace 可追踪、故障证据包可生成。

### Slice G：设备模型与装配闭环

增加 DeviceDefinition Registry、DeviceInstance Binder、Capability/PointMapping Validator、DeviceBindingSnapshot 和 Device Health Coordinator。验收设备实例可复现装配、映射哈希稳定、设备健康状态正确、Recovery/Replay 使用同一 BindingSnapshot。

## 8. 工程质量门禁

| 门禁 | 规则 |
|---|---|
| 架构 | 禁止跨层引用实现程序集 |
| 时间 | 领域代码不得调用 `DateTime.UtcNow` 决定业务结果 |
| 调度 | 只有 Kernel 可以推进 `SimTick` |
| 持久化 | 领域实体不得直接注入 `DbContext` |
| 幂等 | 外部命令必须携带 CommandId + RequestHash；运维命令必须携带 OperationId + ExpectedStateVersion |
| 事件 | 业务事件必须包含 RunId、SimTick、CorrelationId、SchemaVersion |
| 数据 | Command/Transfer/Lock/Outbox 必须符合正式 Schema v2 |
| 恢复 | Snapshot 必须包含输入游标、运行态版本和 Hash |
| 反馈 | Transfer 未 Commit 不得回写 CargoAtDestination |
| 查询 | Event Query/SignalR/Replay 必须使用 EventSequence 游标 |
| 配置 | RunProfile 必须冻结，确定性字段禁止热更新 |
| 审计 | Recovery/Replay/ManualOverride/ConfigRejected 必须有追加审计 |
| Trace | 关键链路节点缺失必须显式返回不完整 |
| 设备装配 | Definition/Config/Capability/PointMapping/ControllerProfile 必须生成一致 BindingSnapshot |
| 健康 | 未 Ready 或处于 Degraded/Recovering 的设备不得执行普通命令 |
| 测试 | 新设备至少通过 Command/Occupancy/Event/Replay/Binding 五类契约测试 |
| 交付 | Architecture/Contract/Migration/Replay/稳定性/Operational Readiness/Device Model 门禁全部通过 |

## 9. 直接工程任务

1. 创建解决方案和项目依赖检查脚本。
2. 实现 `ISimulationRuntimeFactory` 组合根。
3. 为各模块定义 Ports 项目并启用架构测试。
4. 实现 Kernel 单线程运行器、有界输入通道和确定性排序器。
5. 实现命令、Transfer、Outbox 三类事务边界。
6. 实现 PLC 反馈过程映像提交和版本校验。
7. 建立正式 PostgreSQL Schema v2 与 EF Core Migration。
8. 实现 Run Event Query、SignalR 补发和 Replay API。
9. 建立 Slice A/B/C/D/E/F/G 的 CI 门禁。
10. 实现 Snapshot Capture/Restore 和 Replay Hash 校验。
11. 为 `CycleSequence/CommandId/TransferId/OccupancyVersion/ProcessImageVersion` 建立诊断查询索引。
12. 建立 Recovery/Replay 运行态操作审计，确保恢复期间禁止对外完成事件。
13. 实现 RunProfile Schema 校验、配置冻结和热更新拒绝。
14. 实现 `IRunLifecycleCoordinator`、`IOperationalAuditStore` 和 Trace Query。
15. 生成发布证据包：配置摘要、Schema 版本、测试结果、Recovery/Replay Hash、Trace 抽样和审计摘要。
16. 实现 `IDeviceDefinitionRegistry`、`IDeviceInstanceBinder` 和 BindingSnapshot 持久化。
17. 实现 Capability/PointMapping Validator、Device Health Coordinator 和 DM-001～DM-012 测试。
