# Part 03：Kernel 输入背压与恢复策略

## 1. 设计目标

为 HIL、PLC、WCS、API 等异步输入建立有界、可观测、可恢复的入口。背压不改变仿真事实，只决定输入是否延迟、拒绝或进入降级；Kernel 仍是唯一推进 `SimTick` 的执行者。

## 2. 队列分级

| 队列 | 典型输入 | 满载策略 | 是否允许丢弃 |
|---|---|---|---|
| Critical | 急停、故障、安全输出 Ack | 阻断当前 Run 或进入 Degraded | 否 |
| Command | 设备命令、取消、恢复 | 按 CommandId 幂等等待/拒绝 | 否 |
| Telemetry | 诊断、指标、非关键观测 | 丢弃最旧或采样 | 是 |
| Replay | 回放输入 | 失败即终止 | 否 |

每个队列必须暴露 `Depth/Capacity/OldestSimTick/RejectedCount`。

## 3. C# 接口

```csharp
public interface IInputIngress
{
    ValueTask<IngressResult> AcceptAsync(ExternalInput input, CancellationToken ct);
}

public interface IBackpressurePolicy
{
    BackpressureDecision Decide(InputClass inputClass, QueueSnapshot snapshot);
}

public sealed record ExternalInput(
    Guid RunId,
    Guid SessionId,
    long ClockEpoch,
    long ExternalSequence,
    long TargetSimTick,
    InputClass InputClass,
    string Type,
    ReadOnlyMemory<byte> Payload,
    string CorrelationId);
```

## 4. 恢复状态机

```text
Healthy → Saturated → Degraded → Draining → Reconciliating → Healthy
                                  └────────→ Failed
```

- `Saturated`：任一队列达到 80%。
- `Degraded`：Critical/Command 队列达到 95%，或连续拒绝超过阈值。
- `Draining`：暂停新普通输入，保留 Critical 和恢复控制。
- `Reconciliating`：重连后按 Epoch、Sequence、StateVersion 执行对账。
- `Failed`：无法保证事实一致性，禁止自动继续。

## 5. 恢复规则

1. 背压期间不得跳过 Critical 输入。
2. 普通输入被丢弃时必须产生 `InputDropped` 事件，含原因和计数。
3. 断线重连后新 Session 必须递增 `ClockEpoch`。
4. 恢复前先完成状态对账，再开放 Command 队列。
5. Replay 不使用实时背压策略，输入缺失直接失败。

## 6. 验收项

- Critical 输入在队列满载时不得静默丢弃。
- 普通输入丢弃比例和原因可查询。
- 恢复后不存在旧 Epoch 输入覆盖新状态。
- Kernel 不因外部发布或数据库延迟而阻塞。
- 队列深度、拒绝、丢弃、恢复耗时均可被指标采集。