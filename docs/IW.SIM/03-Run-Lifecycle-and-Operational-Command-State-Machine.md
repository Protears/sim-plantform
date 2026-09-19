# Part 03：Run 生命周期与运维命令状态机

## 1. 目标

统一 Run、Pause、Resume、Recover、Replay、Seal 等运维命令的状态转移、权限边界、幂等和审计，避免 API、Worker、Kernel 各自维护不一致的 Run 状态。

## 2. Run 状态

`Draft → Ready → Starting → Running → Pausing → Paused → Recovering → Replaying → Running`

终态：`Sealed | Failed | Archived`

约束：

- `Sealed`、`Archived` 不允许写入业务命令。
- `Recovering`、`Replaying` 不允许发布 Completed、CargoTransferCommitted 等完成事件。
- `Failed` 只能通过显式 `Recovery` 或重新创建 Run 恢复，不允许隐式回到 Running。

## 3. 运维命令接口

```csharp
public enum RunOperation
{
    Start, Pause, Resume, Recover, Replay, Seal, Archive
}

public sealed record RunOperationRequest(
    Guid RunId,
    Guid OperationId,
    RunOperation Operation,
    long ExpectedStateVersion,
    string Reason,
    string CorrelationId);

public interface IRunLifecycleCoordinator
{
    ValueTask<RunOperationResult> ExecuteAsync(
        RunOperationRequest request,
        CancellationToken cancellationToken);
}
```

## 4. 状态转移表

| 当前 | 操作 | 目标 | 前置条件 |
|---|---|---|---|
| Ready | Start | Starting | Profile Frozen、Schema Compatible |
| Starting | StartCompleted | Running | Kernel/Store/Ingress Ready |
| Running | Pause | Pausing | 无活动 Safety Fault |
| Pausing | PauseCompleted | Paused | 当前 Tick 提交完成 |
| Paused | Resume | Running | Snapshot/Queue 校验通过 |
| Running/Paused | Recover | Recovering | SnapshotId 有效 |
| Recovering | Replay | Replaying | Facts/Cursor/Hash 校验通过 |
| Replaying | ReplayCompleted | Running | Replay Hash 一致 |
| Running/Paused | Seal | Sealed | 无活动命令、Outbox 已清空 |
| Sealed | Archive | Archived | 保留策略允许 |

## 5. 幂等规则

1. `OperationId` 是运维命令幂等键。
2. 相同 OperationId + 相同请求摘要返回原结果。
3. 相同 OperationId + 不同请求摘要返回 `RUN-OPS-422`。
4. 状态版本不匹配返回 `RUN-OPS-409`，不得执行部分动作。
5. 状态转移和 `RunOperationAccepted` 审计必须在同一事务边界内提交。

## 6. 错误码

- `RUN-OPS-409`：ExpectedStateVersion 冲突。
- `RUN-OPS-422`：OperationId 请求摘要冲突。
- `RUN-OPS-423`：当前状态不允许该操作。
- `RUN-OPS-503`：Kernel/Store/Recovery 协调器不可用。

## 7. 直接工程任务

- 实现 `IRunLifecycleCoordinator` 和状态转移表驱动校验。
- 为 API、Worker、CLI 统一接入 OperationId 和 ExpectedStateVersion。
- 为每个状态转移增加状态机测试、并发测试和审计测试。
