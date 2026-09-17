# Part14：架构与契约测试实施设计

## 1. 目标

把模块依赖、命令幂等、Transfer 原子性、Outbox 投递、Snapshot/Replay 约束转化为可自动执行的测试门禁。

## 2. 测试项目

```text
Tests/
  ArchitectureTests/
  ContractTests/
  DataMigrationTests/
  RuntimeIntegrationTests/
  ReplayDeterminismTests/
```

## 3. 架构规则

```csharp
public interface IArchitectureRule
{
    string RuleId { get; }
    ArchitectureRuleResult Evaluate(AssemblyCatalog catalog);
}

public sealed record ArchitectureRuleResult(
    string RuleId,
    bool Passed,
    IReadOnlyList<string> Violations);
```

### 必须通过的规则

| RuleId | 约束 |
|---|---|
| ARCH-001 | Domain 不引用 Infrastructure、EF Core、ASP.NET |
| ARCH-002 | Kernel 不引用 API、SignalR、具体数据库实现 |
| ARCH-003 | Device Runtime 不引用 Controller、DbContext |
| ARCH-004 | Contracts 不包含持久化导航属性 |
| ARCH-005 | Application 只能依赖抽象 Port |
| ARCH-006 | HIL/Communication 不得直接写 World State |

## 4. 契约测试

- Command：相同 `RunId + CommandId + RequestHash` 必须返回同一结果。
- Conflict：同一命令不同 RequestHash 必须返回 422。
- Transfer：同一 Cargo 的 Active Transfer 只能一个。
- Occupancy：ExpectedVersion 冲突不得生成 Completed。
- Outbox：业务事务回滚时不得出现可发布消息。
- SignalR：客户端断线补发按 Event Sequence 连续且可去重。
- Replay：同一输入和 Snapshot 必须得到相同 StateHash/ResultHash。

## 5. 测试夹具

```csharp
public sealed class SimulationRunFixture
{
    public required Guid RunId { get; init; }
    public required InMemoryEventStore Events { get; init; }
    public required InMemoryCommandStore Commands { get; init; }
    public required FakeWorldOccupancy Occupancy { get; init; }
    public required DeterministicScheduler Scheduler { get; init; }
}
```

禁止使用随机 Guid 参与排序；测试中所有外部输入必须显式设置 `ExternalSequence` 和 `TargetSimTick`。

## 6. 发布门禁

- PR：架构测试、Schema 校验、单元测试必须通过。
- 联调分支：增加 Transfer 并发、断线补发、Snapshot Restore、Replay Hash。
- 发布候选：100 设备、30ms Tick、72 小时稳定性、无孤儿锁、无重复 CommandId。

## 7. 失败处理

任一强制门禁失败，流水线不得进入后续部署阶段；失败产物必须保留 RunId、CorrelationId、CommandId、TransferId、EventSequence、SnapshotId。
