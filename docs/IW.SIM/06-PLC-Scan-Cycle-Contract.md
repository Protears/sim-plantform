# PLC Runtime 扫描周期契约

## 1. 目标与边界

本文定义 PLC Runtime、Signal/IO Runtime、Simulation Kernel 与 HIL Session 之间的周期边界。PLC Runtime 负责控制器扫描语义，但不拥有仿真时间；Simulation Kernel 是唯一时间权威。HIL 只负责真实控制器接入与帧交换，不直接修改世界状态。

## 2. 周期模型

每个 `PlcTask` 具有独立周期：`PeriodTicks`、`PhaseTicks`、`Priority`。在同一 `SimTick` 内，执行顺序固定为：

```text
InputCommit
  -> PlcTask.ReadProcessImage
  -> LogicExecute
  -> PlcTask.WriteProcessImage
  -> OutputCommit
  -> Device/World Event Publish
```

同一周期内禁止逻辑读取未提交的输入，禁止设备读取未提交的输出。跨 PLC 的输出只在 `OutputCommit` 完成后对外可见。

## 3. C# 工程接口

```csharp
public interface IPlcScanScheduler
{
    ValueTask ScheduleAsync(PlcTaskDefinition task, CancellationToken ct);
    ValueTask<ScanExecutionResult> ExecuteAsync(
        PlcScanContext context, CancellationToken ct);
}

public sealed record PlcTaskDefinition(
    string PlcId,
    string TaskId,
    long PeriodTicks,
    long PhaseTicks,
    int Priority,
    int MaxExecutionTicks);

public sealed record PlcScanContext(
    Guid RunId,
    long SimTick,
    long CycleSequence,
    string InputImageVersion,
    string OutputImageVersion);

public sealed record ScanExecutionResult(
    string PlcId,
    string TaskId,
    long SimTick,
    bool Committed,
    long ExecutionTicks,
    string InputImageHash,
    string OutputImageHash,
    IReadOnlyList<string> Diagnostics);
```

## 4. 调度规则

|规则|约束|
|-|-|
|触发|`(SimTick - PhaseTicks) % PeriodTicks = 0`|
|排序|先 `SimTick`，再 `Priority`，再 `PlcId/TaskId` 字典序|
|超时|超过 `MaxExecutionTicks` 产生 `PLC-SCAN-001`，本周期输出不提交|
|失败|逻辑异常进入 `Faulted`，输出按 PLC 安全策略处理|
|重入|同一 `PlcId+TaskId` 不允许并发执行|
|确定性|调度不得读取 WallClock 或线程调度顺序|

## 5. 状态机

```text
Registered -> Ready -> Executing -> Committing -> Ready
                         |             |
                         v             v
                      Faulted       Degraded
```

`Degraded` 表示本周期完成但诊断超过阈值；`Faulted` 表示本周期没有产生有效输出。

## 6. 与 HIL 的边界

- HIL 输入必须在 `InputCommit` 前完成 `TargetSimTick` 归一化。
- HIL 输出只允许在 `OutputCommit` 后发布。
- 当 HIL 输出需要 Ack 时，Ack 不能阻塞 Kernel；Ack 超时由 HIL Fault Manager 处理。
- Replay 模式忽略原始 WallClock，仅按 `SimTick + ExternalSequence` 重放。

## 7. 可观测性

每个扫描周期必须记录 `RunId、PlcId、TaskId、SimTick、CycleSequence、InputImageHash、OutputImageHash、ExecutionTicks、StateVersion、CorrelationId`。指标：`plc_scan_duration_ticks`、`plc_scan_overrun_total`、`plc_output_commit_total`。

## 8. 测试验收

1. 相同输入和初始状态重复运行 100 次，输出 Hash 完全一致。
2. 两个相同周期 PLC 的排序不依赖线程完成先后。
3. 超时周期不提交部分输出。
4. HIL 迟到输入按 Part12 策略处理，不绕过 InputRecord。
5. 断线期间 Critical Output 使用安全值且不阻塞仿真主循环。
