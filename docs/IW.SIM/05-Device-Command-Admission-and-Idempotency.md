# Part05 设备命令准入、幂等与重试设计

## 1. 设计目标

统一命令从外部入口进入运行时前的准入规则，解决重复提交、版本冲突、过期、重试和取消竞态。准入层只决定“是否允许进入执行队列”，不执行设备算法。

## 2. 准入流水线

```text
Normalize -> ValidateIdentity -> CheckRunState -> IdempotencyLookup
-> ExpectedVersionCheck -> CapabilityCheck -> DeadlineCheck
-> PersistAccepted -> Enqueue
```

任何步骤失败都不得产生 Accepted 事件；只有持久化成功后才允许加入运行时队列。

## 3. C# 契约

```csharp
public sealed record CommandAdmissionResult(
    Guid CommandId,
    bool Accepted,
    DeviceCommandStatus Status,
    string? ErrorCode,
    long? ExistingResultVersion,
    long AdmissionSequence);

public interface ICommandAdmissionService
{
    ValueTask<CommandAdmissionResult> AdmitAsync(
        DeviceCommand command,
        DeviceSnapshot snapshot,
        RunExecutionState run,
        CancellationToken ct);
}

public interface ICommandIdempotencyStore
{
    ValueTask<StoredCommandResult?> FindAsync(Guid runId, Guid commandId, CancellationToken ct);
    ValueTask<bool> TryReserveAsync(Guid runId, Guid commandId, string requestHash, CancellationToken ct);
}
```

## 4. 幂等规则

| 场景 | 处理 |
|---|---|
| 首次 CommandId | 校验并 Reserve，继续准入 |
| 相同 CommandId + 相同 RequestHash | 返回原结果，不重复执行 |
| 相同 CommandId + 不同 RequestHash | 拒绝 `DEV-CMD-422` |
| 同一设备版本已变化 | 拒绝 `DEV-CMD-409` |
| Run 已 Sealed | 拒绝 `RUN-STATE-409` |
| Deadline 小于当前 SimTick | 拒绝 `DEV-CMD-410` |

## 5. 重试与取消

- 传输重试：复用 CommandId，不得创建新业务命令。
- 执行失败重试：只有错误码标记 `Retryable=true` 才允许重新排队。
- 重试必须增加 `Attempt`，但不改变 `CommandId`。
- 取消请求必须使用 `ExpectedCommandVersion`，版本不符返回 `DEV-CMD-409`。
- 已进入不可中断临界区的命令只能返回 CancelRequested，不得伪造 Cancelled。

## 6. 数据结构

建议表 `device_command_dedup`：

```sql
create table device_command_dedup (
  run_id uuid not null,
  command_id uuid not null,
  request_hash char(64) not null,
  status varchar(32) not null,
  result_json jsonb null,
  command_version bigint not null default 1,
  created_at timestamptz not null,
  updated_at timestamptz not null,
  primary key (run_id, command_id)
);
create index ix_device_command_dedup_status on device_command_dedup(run_id, status);
```

## 7. 错误码

- `DEV-CMD-400`：命令格式非法
- `DEV-CMD-409`：设备或命令版本冲突
- `DEV-CMD-410`：命令已过期
- `DEV-CMD-422`：CommandId 对应请求内容不一致
- `DEV-CMD-503`：命令持久化不可用

## 8. 验收项

1. 并发提交同一 CommandId 只有一个 Reserve 成功。
2. 重试不产生第二次设备运动。
3. 不同 RequestHash 必须可诊断地拒绝。
4. 取消与 Executing 边界竞态在 Replay 中结果一致。
