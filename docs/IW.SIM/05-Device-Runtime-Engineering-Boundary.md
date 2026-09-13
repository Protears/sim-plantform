# Part 05：Device Runtime 工程边界与实现基线

## 1. 组件边界

```text
Command API / WCS
        ↓
IDeviceCommandGateway
        ↓
ICommandAdmissionService
        ↓
IDeviceRuntime
   ┌────┼───────────┐
   ↓    ↓           ↓
Capability  Lock   Transfer
Evaluator   Manager Coordinator
        ↓
Simulation Kernel / Event Store
```

Device Runtime 只拥有设备状态演化和设备控制器编排；不直接依赖 EF Core、HTTP、PLC SDK 或 UI。World Model 只拥有位置事实；Kernel 只拥有 SimTick、事件排序和调度。

## 2. C# 接口契约

```csharp
public interface IDeviceRuntime
{
    ValueTask<CommandAdmissionResult> AdmitAsync(
        DeviceCommand command, CancellationToken cancellationToken);

    ValueTask<DeviceExecutionResult> ExecuteAsync(
        DeviceCommand command, DeviceExecutionContext context,
        CancellationToken cancellationToken);

    ValueTask RecoverAsync(
        DeviceRuntimeSnapshot snapshot, CancellationToken cancellationToken);
}

public interface ICommandAdmissionService
{
    ValueTask<CommandAdmissionResult> ValidateAsync(
        DeviceCommand command, CancellationToken cancellationToken);
}

public interface IResourceLockManager
{
    ValueTask<LockAcquireResult> AcquireAsync(
        ResourceLockRequest request, CancellationToken cancellationToken);

    ValueTask RenewAsync(
        LockLease lease, long simTick, CancellationToken cancellationToken);

    ValueTask ReleaseAsync(
        LockLease lease, CancellationToken cancellationToken);
}

public interface IWorldOccupancyTransfer
{
    ValueTask<TransferClaimResult> ClaimAsync(
        TransferIntent intent, CancellationToken cancellationToken);

    ValueTask<TransferCommitResult> CommitAsync(
        TransferCommit commit, CancellationToken cancellationToken);
}
```

## 3. 执行状态机

`Received → Admitted → Locked → Executing → TransferPending → Completed`

异常分支：

- Admitted → Rejected
- Locked → Failed
- Executing → CancelRequested → Cancelled
- TransferPending → Compensating → Compensated
- 任意运行态 → Recovering → Failed

状态变更必须携带 `RunId、CommandId、DeviceVersion、SimTick、CorrelationId`。状态更新采用 ExpectedVersion，禁止无版本覆盖。

## 4. 执行规则

- 命令排序：`TargetSimTick → Priority → CommandId`。
- 锁顺序：`SafetyZone → Track → Device → Cargo → Place`。
- `TransferPending` 期间不得改变最终 Occupancy；只有 Commit 成功后才产生 `CargoOccupancyChanged`。
- Cancel 只取消尚未开始的可取消阶段；进入 Executing 后必须由设备控制器定义安全停止点。
- Runtime 不等待外部 ACK 推进 Kernel；ACK 作为事件输入在下一调度边界处理。

## 5. 可观测性与故障

每个命令必须关联 `CommandId、TransferId、LockId、EventId、CorrelationId、CausationId`。指标包括准入延迟、锁等待、执行耗时、Transfer 冲突、命令重试、恢复耗时。`DEV-CMD-409` 表示版本冲突，`DEV-LOCK-409` 表示资源竞争，`WORLD-OCC-409` 表示位置事实冲突，`RUNTIME-RECOVERY-503` 表示恢复无法继续。

## 6. 工程验收

- 单设备正常执行、失败、取消、重试全部可复现。
- 多设备并发不得出现同一 Cargo 双重执行。
- 任何 Transfer 失败不得产生 Completed 事件。
- Snapshot 恢复后可以从最近未完成阶段继续，且不重复提交 Occupancy。
