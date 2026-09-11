# Part05 设备命令执行契约

## 1. 目标与边界

本契约定义 Application/WCS、Device Runtime、Simulation Kernel、World Model 之间的设备命令提交、排队、执行、完成和失败语义。设备算法仍归 Device Runtime，命令事实与执行状态必须可追踪、可重放。

## 2. 核心原则

1. CommandId 全局唯一；Retry 不生成新业务命令。
2. Device Runtime 只消费已接受命令，不直接读取数据库作为运行时状态。
3. Kernel 负责时间推进与事件排序，Device Runtime 负责能力校验与状态演化。
4. World Model 的 Occupancy 是货载位置唯一事实来源。
5. 取消只允许在 Accepted/Queued 状态，Executing 后只能转为 CancelRequested。

## 3. C# 契约

```csharp
public enum DeviceCommandStatus { Accepted, Queued, Executing, Succeeded, Failed, CancelRequested, Cancelled, Expired }

public sealed record DeviceCommand(
    Guid CommandId,
    Guid RunId,
    Guid DeviceId,
    string CommandType,
    long ExpectedDeviceVersion,
    long TargetSimTick,
    string? CorrelationId,
    string PayloadJson);

public sealed record DeviceCommandResult(
    Guid CommandId,
    DeviceCommandStatus Status,
    long DeviceVersion,
    long CompletedSimTick,
    string? ErrorCode,
    string? ErrorMessage);

public interface IDeviceCommandGateway
{
    ValueTask<DeviceCommandResult> SubmitAsync(DeviceCommand command, CancellationToken ct);
    ValueTask<DeviceCommandResult> CancelAsync(Guid runId, Guid commandId, long expectedVersion, CancellationToken ct);
}
```

## 4. 状态机

`Accepted -> Queued -> Executing -> Succeeded|Failed`；`Accepted|Queued -> Cancelled`；`Executing -> CancelRequested -> Cancelled|Succeeded|Failed`；超过 Deadline 的命令转 `Expired`。

## 5. 一致性与并发

- `(RunId, CommandId)` 唯一。
- 设备级执行锁保证同一 DeviceId 同时只有一个互斥命令。
- ExpectedDeviceVersion 不匹配时拒绝，错误码 `DEV-CMD-409`。
- Kernel 事件排序键：`TargetSimTick, Priority, CommandId`。

## 6. 失败与恢复

设备故障、资源占用、货载不存在、状态版本冲突分别映射 `DEV-CMD-4xx/5xx`。恢复时从 Snapshot 恢复命令队列，仅重放未完成命令；已完成命令由幂等表拦截重复执行。

## 7. 验收项

- 重复提交同一 CommandId 不得产生第二次运动。
- 版本冲突必须可观测且不改变设备状态。
- 取消竞态在 Executing 边界前后结果确定。
- Replay 后 CommandResult 序列与原运行一致。
