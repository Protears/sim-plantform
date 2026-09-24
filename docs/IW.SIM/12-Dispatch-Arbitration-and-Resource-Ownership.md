# Dispatch Arbitration 与资源所有权

## 1. 目标

在 Task/MCS/WCS、Route、Traffic Reservation、DeviceCommand、OccupancyAuthority 之间建立唯一的调度仲裁层，解决多任务竞争同一设备、同一路径、同一货载和同一安全区时的公平性、优先级、抢占和死锁问题。

## 2. 权威边界

- MCS：生成可执行子任务图，不拥有设备锁。
- WCS Dispatcher：提交 `DispatchIntent`，不直接修改 Reservation 或 Occupancy。
- `IDispatchArbiter`：唯一决定是否允许进入 Reservation。
- Traffic Reservation：只负责已获批 Route 的资源预留。
- Device Runtime：只执行已通过 Command Admission 的命令。
- OccupancyAuthority：唯一提交位置事实。

## 3. C# Contract

```csharp
public interface IDispatchArbiter
{
    ValueTask<DispatchDecision> EvaluateAsync(
        DispatchIntent intent,
        DispatchContext context,
        CancellationToken cancellationToken);
}

public sealed record DispatchIntent(
    Guid RunId,
    Guid TaskId,
    Guid SubTaskId,
    Guid CargoId,
    Guid RouteId,
    int Priority,
    DateTimeOffset Deadline,
    string PlanHash,
    string WorldVersion,
    string CorrelationId);

public sealed record DispatchDecision(
    bool Granted,
    string DecisionCode,
    Guid? ArbitrationId,
    int QueueRank,
    DateTimeOffset? RetryAfter,
    string? ConflictOwnerId);
```

## 4. 仲裁顺序

1. Run 状态与 SchemaVersion 校验。
2. Task 依赖和 WorldVersion 校验。
3. Cargo/Task 单活动约束。
4. 安全区与区域冲突检查。
5. 设备能力和当前健康检查。
6. Deadline/优先级/公平配额排序。
7. 生成 `ArbitrationId` 并进入 Reservation。

## 5. 公平性

调度键：`EffectivePriority DESC → Deadline ASC → AgingScore DESC → TaskId ASC`。

`AgingScore = min(WaitingSeconds / AgingIntervalSeconds, AgingCap)`，用于避免低优先级任务永久饥饿。

## 6. 抢占规则

- 只允许抢占尚未进入 `Executing` 的 Reservation。
- 已进入 `TransferPending` 的任务禁止抢占。
- 抢占必须生成 `DispatchPreempted` 和 Audit 记录。
- 被抢占任务必须回到 `Ready`，不得伪造 `Cancelled`。

## 7. 错误码

- `DISPATCH-409-STATE_CONFLICT`
- `DISPATCH-409-RESOURCE_CONFLICT`
- `DISPATCH-423-DEVICE_DEGRADED`
- `DISPATCH-422-WORLD_VERSION_STALE`
- `DISPATCH-429-DEADLINE_MISSED`

## 8. 验收条件

- 任意时刻一个 SubTask 只能有一个有效 Arbitration。
- 同一 Cargo 不得同时获得两个执行资格。
- Reservation 只接受 Granted 的 ArbitrationId。
- 资源冲突与抢占均可通过 EventSequence 重放。
