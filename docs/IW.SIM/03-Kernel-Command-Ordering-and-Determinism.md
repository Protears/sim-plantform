# Part 03：Kernel 命令排序与确定性执行基线

## 1. 目标与边界

本文件定义 Simulation Kernel 如何接收来自 PLC、HIL、WCS、API 和 Device Runtime 的输入，并在同一 `SimTick` 内形成唯一、可重放、可诊断的执行顺序。Kernel 只负责时间、排序、调度和提交边界，不负责设备能力判断、协议解析和数据库映射。

## 2. 输入分层

| 输入层 | 来源 | 允许修改 | 进入方式 |
|---|---|---|---|
| ExternalInput | HIL/PLC/WCS/API | 仅生成意图 | InputRecord → TickBoundary |
| DeviceIntent | DeviceRuntime/Mapper | 生成设备动作 | CommandAdmission → Kernel Queue |
| InternalEvent | Kernel/World/Device | 推进世界状态 | EventScheduler |
| Control | Pause/Resume/Cancel | 改变调度状态 | ControlQueue |

任何外部线程不得直接调用领域实体改变状态。

## 3. 确定性排序键

同一 Run 内使用五元组排序：

```text
(SimTick, Phase, Priority, SourceId, LocalSequence)
```

- `Phase`：`InputCommit < Control < PlcScan < DeviceExecute < OccupancyCommit < EventPublish`
- `Priority`：数字越小优先级越高。
- `SourceId`：使用规范化的稳定 GUID/字符串，不使用线程 ID。
- `LocalSequence`：由输入源在单一 Session 内递增。

禁止使用 wall-clock 到达时间、Task 完成顺序或随机 Hash 作为排序依据。

## 4. C# 可实施接口

```csharp
public interface ISimulationCommandQueue
{
    ValueTask EnqueueAsync(ScheduledCommand command, CancellationToken ct);
    bool TryDequeue(long simTick, out ScheduledCommand command);
}

public sealed record ScheduledCommand(
    Guid RunId,
    long SimTick,
    SimulationPhase Phase,
    int Priority,
    string SourceId,
    long LocalSequence,
    Guid CommandId,
    string PayloadType,
    ReadOnlyMemory<byte> Payload);

public interface IDeterministicScheduler
{
    ValueTask<ScheduleResult> ScheduleAsync(ScheduledCommand command, CancellationToken ct);
    ValueTask<TickExecutionResult> ExecuteTickAsync(long simTick, CancellationToken ct);
}
```

## 5. Tick 提交协议

1. `InputCommit`：冻结本 Tick 输入窗口，拒绝晚于窗口的非 LateInput。
2. `Control`：处理暂停、取消和恢复控制。
3. `PlcScan`：执行固定顺序的 PLC Task。
4. `DeviceExecute`：执行已准入设备命令。
5. `OccupancyCommit`：提交货载/位置事实。
6. `EventPublish`：生成并写入统一事件事实，不等待外部推送完成。

任一阶段失败时，Kernel 只允许按阶段定义的回滚策略处理；不得部分提交后继续推进下一阶段。

## 6. 迟到输入与回放

- `TargetSimTick < CurrentSimTick`：默认拒绝并生成 `LATE_INPUT_REJECTED`。
- `TargetSimTick == CurrentSimTick` 且尚未完成 `InputCommit`：允许进入当前窗口。
- `TargetSimTick > CurrentSimTick`：放入未来 Tick 队列。
- Replay 模式只消费已记录的 InputRecord，不读取网络和 wall-clock。

## 7. 错误码

| 错误码 | 语义 | 处理 |
|---|---|---|
| KERNEL-ORDER-409 | 同一排序键冲突 | 使用 LocalSequence 重排；无法重排则失败 |
| KERNEL-TICK-409 | Tick 已完成仍提交输入 | 拒绝并记录诊断 |
| KERNEL-QUEUE-503 | 有界队列满载 | Critical 阻断；普通输入按策略丢弃 |
| KERNEL-REPLAY-422 | Replay 输入缺失或 Hash 不一致 | 终止 Replay |

## 8. 验收项

- 相同输入集合在 10 次 Replay 中产生相同 Event/State/Result Hash。
- 并行输入到达顺序变化不改变业务结果。
- 一个 Tick 不得出现重复 `InputCommit` 或 `OccupancyCommit`。
- Kernel 主循环不直接调用 EF Core、Socket、PLC SDK。
- 72 小时运行期间不存在不可解释的 Sequence 跳变。