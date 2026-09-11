# Part05 设备资源锁与能力模型

## 1. 设计目标

把“设备能做什么”和“设备当前能否做”分离：Capability 是模型能力，ResourceLock 是运行时占用，避免 WCS、Device Runtime、仿真内核重复判断。

## 2. 能力模型

```csharp
public sealed record DeviceCapability(
    string CapabilityCode,
    string Version,
    IReadOnlySet<string> RequiredInputs,
    IReadOnlySet<string> ProducedOutputs,
    int MaxParallelism,
    TimeSpan? Timeout);

public sealed record ResourceLock(
    Guid LockId,
    Guid RunId,
    Guid ResourceId,
    Guid OwnerCommandId,
    long AcquiredSimTick,
    long LeaseUntilTick,
    int Version);

public interface IDeviceCapabilityRegistry
{
    DeviceCapability GetRequired(Guid deviceId, string commandType);
    bool CanExecute(Guid deviceId, string commandType, DeviceExecutionContext context);
}

public interface IResourceLockManager
{
    ValueTask<ResourceLock?> TryAcquireAsync(Guid resourceId, Guid commandId, long untilTick, CancellationToken ct);
    ValueTask ReleaseAsync(Guid lockId, CancellationToken ct);
}
```

## 3. 锁粒度

- Device：设备互斥执行。
- Track：同一轨道区段互斥占用。
- Cargo：货载搬运互斥。
- Place：库位写入互斥。
- SafetyZone：安全区进入互斥。

锁的所有权只属于 CommandId，不允许模块间“借用”锁。

## 4. 生命周期

`Requested -> Acquiring -> Held -> Renewing -> Released`；Tick 超过 LeaseUntilTick 自动进入 `Expired`，设备进入 Degraded 并产生事件 `ResourceLockExpired`。

## 5. 死锁与顺序

锁按固定顺序申请：`SafetyZone -> Track -> Device -> Cargo -> Place`。无法一次获取时全部释放并按确定性退避重新排队；禁止部分持有等待。

## 6. 错误码

- `DEV-CAP-404`：命令能力不存在。
- `DEV-CAP-422`：输入/输出契约不满足。
- `DEV-LOCK-409`：资源被占用。
- `DEV-LOCK-408`：锁租约超时。
- `DEV-LOCK-500`：锁状态持久化失败。

## 7. 验收

- 同一 Cargo 不得被两个活动命令同时占用。
- 资源锁申请顺序跨线程、跨回放一致。
- 任一锁获取失败不得留下孤儿锁。
- Snapshot/Restore 后锁状态与未完成 Command 一致。
