# Part 14：工程实施基线与模块装配规则

## 1. 目标与适用范围

本文件把 Part02/05/06/07/08/10/11/12 的设计契约收敛为 .NET 10 模块装配基线，作为编码、代码评审、测试和部署的共同入口。它不重新定义领域规则，而是规定：程序集边界、依赖方向、线程模型、端口适配、事务边界、运行时装配和最小可交付切片。

配套实施文档：

- `14-Project-Structure-and-Dependency-Verification.md`
- `10-Command-Occupancy-Transaction-Implementation.md`
- `11-Command-Api-SignalR-Delivery-Contract.md`
- `06-PLC-Feedback-Process-Image-Contract.md`
- `14-Engineering-Slice-Acceptance-Matrix.md`
- `ADR/ADR-016-Commit-Then-Publish-Facts.md`

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
```

依赖规则：

- Domain 只依赖 BCL；禁止依赖 EF Core、ASP.NET、SignalR、PLC SDK。
- SimulationKernel 依赖 Domain，不依赖 Api、Data；通过端口读取输入和写入事件。
- DeviceRuntime、PlcRuntime、SignalIo 依赖 Domain 与明确的 Application Ports，不互相引用实现程序集。
- Communication/Hil 只能通过 Ports 向 Application/SignalIo 提供帧和信号；禁止直接写 World State。
- Data 实现 Application/Domain 定义的 Store 接口；EF Core 只出现在 Data。
- Api/Worker 负责组合根和生命周期，不在控制器中执行设备算法。

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

## 4. 线程与调度模型

- Kernel 主循环是唯一推进 SimTick 的线程。
- PLC 逻辑、设备状态演化和 Occupancy Commit 必须在 Kernel 调度上下文内完成；异步 I/O 只能投递输入事件，不得直接改变领域状态。
- Communication/HIL 使用独立 I/O 线程或 async socket，但通过有界 Channel 将数据交给 InputIngress。
- Snapshot/Outbox 持久化为后台消费者；持久化延迟不能阻塞 Kernel，但必须提供 `PersistenceLag` 和降级门槛。
- 同一 Run 的业务事件按 `(SimTick, Priority, Sequence)` 排序；不得依赖线程完成顺序。

## 5. 最小工程切片

### Slice A：纯仿真

Kernel + SignalIo + DeviceRuntime + WorldModel + EventStore。验收 PLC/HIL 关闭时，普通输送机完成货载转移并生成可重放事件。

### Slice B：PLC 联调

增加 PlcRuntime + ProcessImage + S7/Modbus Adapter。验收 PLC 输出驱动设备，设备传感器反馈回 PLC，周期 Hash 稳定。

### Slice C：HIL

增加 Communication + HilSession + TimeNormalizer + Ack/SafetyGuard。验收断线、迟到输入、旧 Epoch、Critical Output Ack 超时。

## 6. 工程质量门禁

| 门禁 | 规则 |
|---|---|
| 架构 | 禁止跨层引用实现程序集 |
| 时间 | 领域代码不得调用 `DateTime.UtcNow` 决定业务结果 |
| 持久化 | 领域实体不得直接注入 `DbContext` |
| 幂等 | 外部命令必须携带 CommandId + RequestHash |
| 事件 | 业务事件必须包含 RunId、SimTick、CorrelationId、SchemaVersion |
| 恢复 | Snapshot 必须包含输入游标、运行态版本和 Hash |
| 测试 | 新设备至少通过 Command/Occupancy/Event/Replay 四类契约测试 |
| 交付 | Architecture/Contract/Migration/Replay/稳定性门禁全部通过 |

## 7. 直接工程任务

1. 创建解决方案和项目依赖检查脚本。
2. 实现 `ISimulationRuntimeFactory` 组合根。
3. 为各模块定义 Ports 项目并启用架构测试。
4. 实现 Kernel 单线程运行器和有界输入通道。
5. 实现命令、Transfer、Outbox 三类事务边界。
6. 实现 PLC 反馈过程映像提交和版本校验。
7. 建立 Slice A/B/C 的 CI 门禁。
