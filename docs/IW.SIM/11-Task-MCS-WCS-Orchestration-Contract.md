# Task/MCS/WCS 编排契约

## 1. 目标

定义 Task、MCS、WCS、Device Runtime、Route、Traffic Reservation、Occupancy Transfer 的责任边界，避免 WCS 直接操控设备、MCS 直接修改位置事实或 Task 携带运行时设备状态。

## 2. 领域边界

- **Task**：业务目标与约束，不保存设备执行细节。
- **MCS**：任务拆分、优先级、依赖、批量编排和失败策略。
- **WCS**：把可执行子任务映射为 Route/Reservation/DeviceCommand。
- **Device Runtime**：执行设备能力，不决定业务任务优先级。
- **World/Occupancy**：唯一位置事实权威。
- **Route/Traffic**：只提供可执行路径和资源占用，不修改货载位置。

## 3. 任务模型

```csharp
public sealed record TaskSpec(
    Guid TaskId,
    Guid RunId,
    string TaskType,
    Guid? CargoId,
    Guid? SourcePlaceId,
    Guid? TargetPlaceId,
    int Priority,
    IReadOnlyList<Guid> DependsOn,
    string RequestHash,
    long ExpectedWorldVersion);

public sealed record TaskExecution(
    Guid TaskId,
    string State,
    Guid? ActiveSubTaskId,
    Guid? RouteId,
    Guid? ReservationId,
    Guid? CommandId,
    long StateVersion);
```

## 4. 状态机

`Created → Planned → Ready → Reserving → Reserved → Dispatching → Executing → TransferPending → Completed`

失败路径：

- `PlanningFailed`
- `ReservationRejected`
- `DispatchFailed`
- `ExecutionFailed`
- `Compensating`
- `Cancelled`
- `Expired`

进入 `Completed` 的唯一条件：`DeviceExecutionSucceeded + OccupancyTransferCommitted + RequiredFeedbackPublished`。

## 5. 编排时序

```text
TaskSubmitted
  → MCS validates dependency/capability
  → WCS resolves Route
  → Traffic reserves SafetyZone/Region/Track/Place
  → WCS creates DeviceCommand
  → DeviceRuntime executes
  → OccupancyAuthority commits Transfer
  → ProcessImage feedback published
  → TaskCompleted event
```

## 6. 幂等与并发

- `(RunId, TaskId)` 是任务业务主键。
- `(RunId, TaskId, RequestHash)` 负责重复提交判定。
- `ExpectedWorldVersion` 防止任务基于旧 Occupancy 规划。
- 同一 Cargo 同时只能存在一个 `Executing/TransferPending` 任务。
- 同一目标 Place 的活动任务必须通过 Traffic Reservation 竞争。

## 7. C# 接口

```csharp
public interface ITaskOrchestrator
{
    TaskPlanResult Plan(TaskSpec spec, SimulationContext context);
    TaskDispatchResult Dispatch(Guid taskId, long expectedStateVersion);
    TaskCancelResult Cancel(Guid taskId, string reason);
}

public interface IMcsPlanner
{
    IReadOnlyList<TaskSpec> Expand(TaskSpec task, PlanningContext context);
}

public interface IWcsDispatcher
{
    DispatchPlan Build(TaskSpec task, RouteResolutionResult route,
        TrafficReservationResult reservation);
}
```

## 8. 错误码

- `TASK-409`：Task 状态版本冲突
- `TASK-422`：Task 约束不合法
- `TASK-423`：Cargo 已被其他任务占用
- `MCS-409`：依赖任务未完成
- `WCS-409`：Route/Reservation 竞争
- `WCS-422`：无可执行设备能力
- `TASK-RECOVERY-503`：无法恢复任务执行

## 9. 禁止规则

1. WCS 不得直接写 Occupancy。
2. MCS 不得直接调用 Device Runtime。
3. Task 不得保存瞬时传感器值作为完成事实。
4. DeviceCommand 完成不得单独导致 TaskCompleted。
5. Replay 不得读取当前任务配置替代原始 TaskSnapshot。

## 10. 验收项

- 依赖图存在环时拒绝规划。
- 旧 WorldVersion 规划结果不得进入 Dispatch。
- Reservation 失败不得生成 DeviceCommand。
- 同一 Task 重试返回同一 PlanHash。
- Transfer 未 Commit 不得发布 TaskCompleted。
- Task 取消必须释放 Reservation 和未执行 Command。
