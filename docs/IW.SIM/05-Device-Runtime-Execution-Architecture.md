# Part05 设备运行时执行架构

## 1. 目标与审查结论

本章把设备命令、能力校验、资源锁、运动状态、货载接驳和事件事实链收敛为可实施运行时。边界如下：

- Application/WCS 只提交意图，不驱动设备内部状态机。
- Device Runtime 只在内存 FrozenState 上执行，不在仿真热路径读取数据库。
- Simulation Kernel 是唯一的 SimTick 推进者和事件排序者。
- World Model 只负责空间、位置和占用事实，不负责设备动作决策。

## 2. 运行时组件

```text
DeviceCommandGateway
  -> CommandAdmission
  -> CapabilityEvaluator
  -> ResourceLockManager
  -> DeviceController
  -> Motion/Transfer Model
  -> OccupancyCoordinator
  -> DeviceEventPublisher
```

### 2.1 组件职责

| 组件 | 输入 | 输出 | 不负责 |
|---|---|---|---|
| CommandAdmission | DeviceCommand | Accepted/Rejected | 设备运动 |
| CapabilityEvaluator | Command + DeviceSnapshot | CapabilityDecision | 锁管理 |
| ResourceLockManager | LockRequest | LockLease | 货载移动 |
| DeviceController | AcceptedCommand + FrozenState | DeviceStateDelta | 数据库写入 |
| OccupancyCoordinator | TransferIntent | OccupancyCommit | 协议通信 |
| EventPublisher | Facts | EventEnvelope | 改写事实 |

## 3. C# 接口

```csharp
public interface IDeviceRuntime
{
    ValueTask<CommandAdmissionResult> AdmitAsync(
        DeviceCommand command, RuntimeContext context, CancellationToken ct);

    DeviceExecutionResult Execute(
        AcceptedDeviceCommand command,
        DeviceRuntimeState state,
        SimTick tick);
}

public interface IDeviceController
{
    DeviceExecutionResult Execute(
        AcceptedDeviceCommand command,
        DeviceRuntimeState state,
        SimTick tick);
}

public interface ICapabilityEvaluator
{
    CapabilityDecision Evaluate(
        DeviceCommand command,
        DeviceDescriptor descriptor,
        DeviceRuntimeState state);
}

public interface IOccupancyCoordinator
{
    OccupancyCommitResult Commit(
        TransferIntent intent,
        WorldOccupancySnapshot snapshot,
        SimTick tick);
}
```

## 4. 执行状态机

`Received -> Admitted -> Locked -> Executing -> TransferPending -> Committed -> Completed`。

异常转移：

- `Received -> Rejected`
- `Admitted -> Expired`
- `Locked -> LockLost`
- `Executing -> Faulted`
- `TransferPending -> OccupancyConflict`
- `Faulted|OccupancyConflict|LockLost -> Recovering -> Requeued|Failed`

状态转换要求：每次转换生成 `DeviceExecutionChanged`，并递增 `DeviceVersion`；禁止跨越中间状态直接写入 Completed。

## 5. 关键不变量

1. 一个活动命令最多持有一个设备执行上下文。
2. 一个 CargoId 同时只能存在一个 ActiveTransfer。
3. `TransferPending` 未提交前，设备运动结果不能修改 WorldOccupancy。
4. `OccupancyCommit` 失败时，设备必须进入可恢复状态，不能伪造完成事件。
5. 所有状态演化使用 `SimTick -> Priority -> CommandId` 稳定排序。

## 6. 失败恢复

- 锁租约到期：停止新动作，发布 `ResourceLockExpired`，转 Recovering。
- 货载冲突：保留设备状态，回滚未提交 TransferIntent，命令转 Failed 或 Requeued。
- 进程恢复：从 Snapshot 恢复 DeviceRuntimeState、活动命令、锁和 OccupancyVersion。
- 事件重投：通过 `(RunId, EventId)` Inbox 去重，不重复执行设备控制器。

## 7. 可观测性

必须记录：`RunId, DeviceId, CommandId, CargoId, LockId, SimTick, DeviceVersion, OccupancyVersion, CorrelationId, CausationId`。指标：命令等待时间、锁等待时间、执行周期、占用冲突率、恢复次数、设备状态 Hash。

## 8. 验收

- 同一设备并发命令只有一个进入 Locked。
- Occupancy 冲突不会生成 Completed。
- Snapshot 恢复后重新运行，Device/Occupancy Hash 与原运行一致。
- 设备控制器不依赖 EF Core、HTTP、PLC SDK。
