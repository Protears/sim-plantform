# Part 14：工程项目结构与依赖验证

## 1. 目标

将程序集边界转化为可自动执行的项目结构和依赖验证规则，阻止 Domain、Kernel、Runtime、Data、Api 之间出现反向引用。

## 2. 项目分层

```text
Logistics.Simulation.Domain
Logistics.Simulation.Contracts
Logistics.Simulation.Application
Logistics.Simulation.SimulationKernel
Logistics.Simulation.DeviceRuntime
Logistics.Simulation.PlcRuntime
Logistics.Simulation.SignalIo
Logistics.Simulation.Communication
Logistics.Simulation.Hil
Logistics.Simulation.Data
Logistics.Simulation.Api
Logistics.Simulation.Worker
```

`Contracts` 只存跨模块 DTO/Event/枚举，不允许包含数据库实体和业务规则。

## 3. 依赖矩阵

| 项目 | 可依赖 | 禁止依赖 |
|---|---|---|
| Domain | BCL、Contracts | EF Core、ASP.NET、SignalR、PLC SDK |
| Kernel | Domain、Contracts | Data、Api、具体协议适配器 |
| DeviceRuntime | Domain、Contracts、Application Ports | DbContext、Controller |
| PlcRuntime | Domain、Contracts、SignalIo Ports | WorldState 持久化实现 |
| Data | Domain、Contracts、Application Ports、EF Core | Api、UI |
| Api/Worker | Application、Data、所有组合根 | 直接执行设备算法 |

## 4. 可执行规则

```csharp
public interface IArchitectureRule
{
    string RuleId { get; }
    ArchitectureRuleResult Evaluate(ProjectGraph graph);
}

public sealed record ArchitectureRuleResult(
    string RuleId,
    bool Passed,
    IReadOnlyList<string> Violations);
```

规则至少包括：`ARCH-DOMAIN-001`、`ARCH-KERNEL-002`、`ARCH-DATA-003`、`ARCH-API-004`。

## 5. CI 门禁

- 生成项目引用图并与白名单比对。
- 发现禁止引用直接失败，不允许 warning。
- 每次新增程序集必须补充依赖矩阵和对应架构测试。
- 失败输出项目、引用方、被引用方、规则编号和修复建议。

## 6. 验收项

1. Domain 引用 EF Core 时构建失败。
2. Kernel 引用 Data 时架构测试失败。
3. Api 直接调用 `IDeviceController` 实现类时测试失败。
4. Contracts 不出现 `DbContext`、实体导航属性或线程同步原语。
5. 所有项目均能从组合根启动并通过依赖验证。
