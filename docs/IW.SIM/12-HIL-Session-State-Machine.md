# Part12 HIL - Session 状态机与命令契约

## 状态转换表

| From | Command/Event | To | 失败处理 |
|---|---|---|---|
| Created | Connect | Connecting | 记录失败并 Failed |
| Connecting | Connected | Handshaking | 超时 -> Failed |
| Handshaking | Validated | Ready | 合约不匹配 -> Failed |
| Ready | Start | Running | Run 未 Running 则拒绝 |
| Running | ConnectionLost | Degraded | 启动恢复窗口 |
| Degraded | Reconnected | Recovering | 重建协议状态 |
| Recovering | Resynchronized | Running | 失败 -> Failed |
| Running | Pause | Pausing -> Paused | 超时 -> Failed |
| Paused | Resume | Running | 需重新验证连接 |
| Any Active | Stop | Stopping -> Stopped | Fail-Safe 优先 |

## Command Contract

```csharp
public sealed record StartHilSessionCommand(
    Guid RunId,
    string IntegrationProfileId,
    string IoContractVersion,
    string ExpectedProfileHash);

public sealed record HilCommandResult(
    Guid SessionId,
    HilSessionState State,
    string? ErrorCode,
    long StateVersion);
```

所有状态命令必须携带或基于 `StateVersion` 实现乐观并发控制。重复 Start 使用 IdempotencyKey：同一 Run + 同一 Profile + 同一请求键返回已有结果，不创建第二个 Session。

## 错误码

```text
HIL_RUN_NOT_ACTIVE
HIL_SESSION_CONFLICT
HIL_CONNECT_TIMEOUT
HIL_HANDSHAKE_FAILED
HIL_CONTRACT_MISMATCH
HIL_HEARTBEAT_TIMEOUT
HIL_RESYNC_FAILED
HIL_OUTPUT_ACK_TIMEOUT
HIL_UNSAFE_OUTPUT_BLOCKED
```

## 验收测试

1. 并发 Start 仅成功一个 Session；
2. Ready 之前 Start Simulation 被拒绝；
3. 重复 Command 返回幂等结果；
4. ConnectionLost 必须进入 Degraded；
5. Recovering 完成前禁止普通 Output；
6. Stop 在任意 Active 状态最终达到 Stopped 或明确 Failed。
