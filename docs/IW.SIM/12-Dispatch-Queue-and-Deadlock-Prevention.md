# Dispatch Queue、资源等待与死锁预防

## 1. 资源图

每个等待请求形成边 `Task → Resource`，已持有资源形成 `Resource → Task`。调度器每个 Arbitration 周期必须检测有向环。

## 2. 固定资源排序

所有资源按以下顺序申请：

`SafetyZone → Region → Track → Place → Device → PlcSession`

禁止反向申请；跨设备任务必须先计算完整资源集合，再一次性提交 Reservation。

## 3. 状态机

`Queued → Inspecting → WaitingResource → Granted → Executing → Released`

异常：`DeadlockSuspected → VictimSelected → Preempting → Requeued`。

## 4. 受害者选择

按以下键选择受害任务：

`LowestEffectivePriority → LargestRemainingWork → LatestDeadline → HighestTaskId`

禁止选择已进入 `TransferPending` 或 SafetyCritical 的任务。

## 5. C# Contract

```csharp
public interface IDeadlockDetector
{
    ValueTask<DeadlockReport> DetectAsync(
        IReadOnlyCollection<ResourceWaitEdge> edges,
        CancellationToken cancellationToken);
}

public sealed record ResourceWaitEdge(string From, string To, string ResourceId);
public sealed record DeadlockReport(bool HasCycle, IReadOnlyList<string> CycleNodes, string GraphHash);
```

## 6. 运行规则

- 检测到环后不再发放新的同图 Reservation。
- 受害任务必须产生 `DeadlockVictimSelected`。
- 资源释放和任务重排必须在同一 Kernel Tick 的 Commit 阶段完成。
- 重排后的 PlanHash 必须变化并进入审计。

## 7. 可验证约束

- 资源申请顺序逆序时架构测试失败。
- 周期图中任何环必须可重放。
- 同一任务最多一个 VictimSelection。
- Deadlock 处理不能直接修改 Occupancy。
