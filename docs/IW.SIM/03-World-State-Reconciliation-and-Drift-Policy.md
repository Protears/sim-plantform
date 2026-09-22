# Part 03：World State Reconciliation 与 Drift Policy

## 1. 目标

定义 Device Runtime、PLC Process Image、Occupancy、Route Reservation、Snapshot 之间的对账、漂移检测和恢复策略。任何“观测到的状态”不能直接覆盖 World Fact。

## 2. 漂移分类

| Drift | 示例 | 处理 |
|---|---|---|
| D1 观测延迟 | 传感器晚一个周期 | 等待窗口，不改事实 |
| D2 版本落后 | ProcessImageVersion 旧 | 丢弃并记录 |
| D3 位置冲突 | 设备说到位但 Occupancy 未 Commit | 阻断完成反馈，进入 Reconciliation |
| D4 结构冲突 | Binding/Layout Hash 不同 | Run Failed |
| D5 安全冲突 | SafetyZone 与锁不一致 | 立即停止相关命令 |

## 3. C# 契约

```csharp
public interface IWorldReconciliationService
{
    ValueTask<ReconciliationDecision> EvaluateAsync(
        ReconciliationInput input,
        CancellationToken cancellationToken);

    ValueTask<ReconciliationResult> ApplyAsync(
        ReconciliationDecision decision,
        CancellationToken cancellationToken);
}

public sealed record ReconciliationInput(
    Guid RunId,
    Guid DeviceId,
    long SimTick,
    long OccupancyVersion,
    long ProcessImageVersion,
    string LayoutHash,
    string BindingHash,
    IReadOnlyList<ObservedFact> Facts);
```

## 4. 决策状态机

```text
Observed
  → Correlated
  → Consistent
  → Reconciled
  → Resumed

Observed → DriftDetected → Degraded
DriftDetected → CriticalConflict → Failed
```

- `Consistent`：仅当版本、快照哈希、锁和 Occupancy 关系全部满足。
- `Reconciled`：形成明确的事实修复记录，不允许静默覆盖。
- `Degraded`：只允许安全命令和诊断命令。

## 5. 规则

1. Observation 不得直接写 Occupancy。
2. `CargoAtDestination=true` 只能由成功 Transfer Commit 推导。
3. D2 只丢弃，不触发状态回退。
4. D3 必须阻止 Completed 事件并创建 Reconciliation 记录。
5. D4 立即阻止 Recovery/Replay 继续执行。
6. 所有人工修复必须有 `OperationId`、前后 Hash 和 Audit。

## 6. 验收

- 注入旧 ProcessImageVersion 后，World Fact 不变。
- 注入位置冲突后，Run 进入 Degraded 或 Failed，不能继续发布完成事件。
- 对账完成后 Hash、OccupancyVersion、EventSequence 可追溯。
