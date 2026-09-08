# Part12 HIL - 时间同步与 Tick 对齐

## 1. 基本原则

HIL 同时存在墙上时钟和 SimClock：

```text
WallClock: 外部 PLC/网络发生时间
SimClock : 仿真确定性时间
```

两者不能互相替代。外部输入首先记录接收时间、外部序号和延迟估计，再由 Synchronizer 映射到确定的目标 Tick。

## 2. 对齐模型

```csharp
public sealed record HilInputStamp(
    long ExternalSequence,
    DateTimeOffset ReceivedAt,
    DateTimeOffset? SourceTimestamp,
    long TargetSimTick,
    double EstimatedLatencyMs,
    long ClockEpoch);
```

映射：

```text
TargetSimTick = max(CurrentTick + MinInputLeadTicks,
                    EstimateTick(ReceivedAt, ClockEpoch))
```

禁止把网络到达顺序直接作为同 Tick 内的执行顺序。排序规则为：

```text
(TargetSimTick, InputPriority, ExternalSequence, StableSequence)
```

## 3. 同步策略

- 运行开始建立 ClockEpoch；
- 周期性采样 RTT/Jitter；
- 仅允许渐进修正映射参数，禁止突然修改历史 Tick；
- Jitter 超阈值触发 Degraded；
- 延迟超过 MaxLateInputWindow 的输入进入 LateInput 策略。

LateInput 策略：

```text
Reject       严格联调
ApplyNextTick 实时优先
PauseAndInspect 调试模式
```

由 RunProfile 显式配置，默认严格模式禁止静默重写历史。

## 4. 输出节拍

Output Profile 支持：

```text
OnChange
FixedCycle
TickBoundary
EdgePreserving
```

关键脉冲必须声明最小保持 Tick，避免 PLC 扫描周期漏读。

## 5. 指标与测试

指标：RTT、Jitter、LateInput、TickOffset、OutputLag。

测试：稳定网络、抖动、乱序、重复、时钟跳变、长暂停恢复。验收要求是同一录制 InputSet Replay 时不依赖原始墙钟。
