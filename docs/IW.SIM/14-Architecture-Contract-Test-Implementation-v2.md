# Part 14：架构与契约测试实施基线 v2

## 1. 测试项目

```text
tests/
  Logistics.Simulation.ArchitectureTests/
  Logistics.Simulation.ContractTests/
  Logistics.Simulation.DataMigrationTests/
  Logistics.Simulation.RuntimeIntegrationTests/
  Logistics.Simulation.ReplayDeterminismTests/
```

## 2. 架构规则

| 编号 | 规则 | 失败级别 |
|---|---|---|
| ARCH-001 | Domain 不得引用 EF Core/ASP.NET/SignalR/PLC SDK | 阻断 |
| ARCH-002 | Kernel 不得引用 Api/Data 具体实现 | 阻断 |
| ARCH-003 | DeviceRuntime 不得持有 DbContext | 阻断 |
| ARCH-004 | Communication/HIL 不得直接写 World State | 阻断 |
| ARCH-005 | Event Sequence 只能由 EventStore 分配 | 阻断 |
| ARCH-006 | Replay 不得写原始 Run | 阻断 |

## 3. C# 结构化接口

```csharp
public interface IArchitectureRule
{
    string Id { get; }
    ArchitectureRuleResult Evaluate(AssemblyGraph graph);
}

public sealed record ArchitectureRuleResult(
    string RuleId,
    bool Passed,
    IReadOnlyList<string> Violations);

public interface IContractScenario<TRequest, TResponse>
{
    Task<TResponse> ExecuteAsync(TRequest request, CancellationToken cancellationToken);
    Task AssertInvariantAsync(TResponse response, CancellationToken cancellationToken);
}
```

## 4. 强制契约测试

- Command：重复 CommandId、RequestHash 冲突、ExpectedVersion 冲突、Sealed Run 拒绝。
- Transfer：Cargo 互斥、Destination 互斥、版本冲突、Commit 原子性。
- Outbox：发布重试、租约接管、DeadLetter、不得重复执行设备动作。
- Feedback：旧 ProcessImageVersion、旧 Epoch、Quality=Bad、Transfer 未 Commit。
- Replay：输入顺序变化、Snapshot 恢复、Hash 一致性、ReplayRun 隔离。

## 5. CI 门禁

```yaml
stages:
  - architecture
  - contract
  - migration
  - integration
  - replay
  - stability

gates:
  architecture: required
  contract: required
  migration: required
  replay_determinism: required
  stability_72h: required
```

任何 required gate 失败不得进入 HIL 联调分支。测试失败必须保留 RunId、CommandId、TransferId、EventSequence、CorrelationId、SnapshotId。

## 6. 失败分类

- 配置错误：阻断启动。
- 契约错误：拒绝单条输入并生成诊断事件。
- 事实冲突：当前命令失败，Run 可降级。
- 恢复错误：进入 Recovery/Failed，不得继续发布完成事件。

## 7. 直接实施任务

1. 使用 NetArchTest 或自定义 AssemblyGraph 实现 ARCH-001～006。
2. 为 T1/T2/T3 事务建立共享测试夹具。
3. 建立 PostgreSQL Testcontainer 执行 MigrationTests。
4. 建立 deterministic scheduler 双输入顺序测试。
5. 在 CI 中将 required gate 设置为发布阻断。