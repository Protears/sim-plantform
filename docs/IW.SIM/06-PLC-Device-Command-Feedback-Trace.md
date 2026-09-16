# Part 06：PLC—设备命令—反馈链路追踪

## 1. 目标

将 `CycleSequence`、`CommandId`、`TransferId`、`OccupancyVersion` 串成一条可查询链路，解决 PLC 输出已提交但设备未执行、设备完成但 PLC 未收到反馈时无法定位的问题。

## 2. 关联模型

```text
PlcCycle(CycleSequence)
  └─ DeviceCommand(CommandId)
      └─ Transfer(TransferId)
          └─ OccupancyCommit(OccupancyVersion)
              └─ PlcFeedback(ProcessImageVersion)
```

每个节点必须记录 `RunId`、`CorrelationId` 和 `SchemaVersion`。

## 3. DTO/Event Contract

```csharp
public sealed record DeviceCommandTrace(
    Guid RunId,
    long CycleSequence,
    Guid CommandId,
    Guid? TransferId,
    long? OccupancyVersion,
    long? ProcessImageVersion,
    string Status,
    string CorrelationId);

public sealed record PlcFeedbackPublished(
    Guid RunId,
    long CycleSequence,
    Guid CommandId,
    long ProcessImageVersion,
    string FeedbackHash,
    string Quality,
    long SimTick);
```

## 4. 状态一致性规则

- Command 未进入 `Executing`，不得写入完成反馈。
- Transfer 未 `Committed`，不得输出 `CargoAtDestination=true`。
- `OccupancyVersion` 必须等于 Feedback 中记录的版本，否则反馈标记为 `Stale`。
- 同一 `CycleSequence` 只能有一个 `Committed` Feedback。
- Feedback 重试不重复生成设备命令。

## 5. 诊断查询

必须支持按以下任一键反查完整链路：

- `CommandId`
- `TransferId`
- `CycleSequence`
- `OccupancyVersion`
- `CorrelationId`

查询结果必须返回各阶段时间、状态、错误码和最后事实事件。

## 6. 验收项

- PLC 输出到设备完成反馈可在单次查询中闭环展示。
- Transfer Commit 失败时 PLC 不得收到完成信号。
- 重复 Feedback 不会生成第二次 Command。
- 旧 ProcessImageVersion 不得覆盖新反馈。