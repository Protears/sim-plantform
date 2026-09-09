# Part 08：通信架构与 HIL 集成边界

## 1. 目标与审查结论

本章补齐通信层与 HIL 会话、Signal/IO Runtime、PLC Runtime 之间的工程边界。通信层只负责“连接、协议、编解码、传输质量”，不拥有仿真时钟、设备状态或安全策略；HIL Session 负责会话编排与时序归一化；Signal/IO Runtime 负责信号语义与映射；Simulation Kernel 只接收已验证的输入事件。

本次审查关闭的主要缺口：连接状态与 HIL 状态混用、协议适配器直接写入设备状态、重连后重复输入未定义、输出确认责任不清、错误码跨章节不一致。

## 2. 模块边界与依赖规则

```text
IProtocolAdapter -> IHilTransport -> IHilSession -> IInputIngress/IOutputEgress
                                              -> Signal/IO Runtime
                                              -> Part10 Event/Input Store
```

强制规则：

1. `IProtocolAdapter` 不得引用 `SimulationKernel`、`DeviceRuntime`、EF Core。
2. `IHilTransport` 只返回字节帧与传输元数据，不解释业务信号。
3. `IHilSession` 负责握手、心跳、时钟偏差、序列号与会话代次 `ClockEpoch`。
4. 输入只有经过 Contract 校验、去重、时间归一化后，才能进入 Part10 的 `ExternalInputRecord`。
5. 输出由 Signal/IO Runtime 产生，HIL 层只负责安全门控、编码、发送和确认跟踪。

## 3. 核心接口

```csharp
public interface IProtocolAdapter
{
    string ProtocolType { get; }
    ValueTask<ProtocolHandshakeResult> HandshakeAsync(
        ReadOnlyMemory<byte> payload, CancellationToken cancellationToken);
    ValueTask<ProtocolDecodeResult> DecodeAsync(
        ReadOnlyMemory<byte> payload, CancellationToken cancellationToken);
    ValueTask<ReadOnlyMemory<byte>> EncodeAsync(
        HilSignalFrame frame, CancellationToken cancellationToken);
}

public interface IHilTransport
{
    HilTransportState State { get; }
    ValueTask ConnectAsync(HilEndpoint endpoint, CancellationToken cancellationToken);
    ValueTask DisconnectAsync(string reason, CancellationToken cancellationToken);
    IAsyncEnumerable<HilRawFrame> ReadFramesAsync(CancellationToken cancellationToken);
    ValueTask SendAsync(HilRawFrame frame, CancellationToken cancellationToken);
}

public interface IHilSession
{
    Guid SessionId { get; }
    long StateVersion { get; }
    ValueTask<HilSessionSnapshot> GetSnapshotAsync(CancellationToken cancellationToken);
    ValueTask StartAsync(StartHilSessionCommand command, CancellationToken cancellationToken);
    ValueTask StopAsync(StopHilSessionCommand command, CancellationToken cancellationToken);
}
```

## 4. 输入与输出主流程

输入流程：Transport 收帧 → Adapter 解码 → Contract 校验 → `ExternalSequence` 去重 → `TargetSimTick` 归一化 → 记录 InputRecord → 在 TickBoundary 注入 Kernel。

输出流程：Runtime 状态变化 → SafetyGuard → 按 OutputProfile 编码 → Transport 发送 → 等待 Ack → 写入 `HilOutputAcknowledged` 或 `HilOutputAckTimeout`。

禁止在 Transport 线程直接调用设备动作；禁止用墙上时钟覆盖仿真时钟；禁止把 Ack 当作状态事实，Ack 只证明对端收到。

## 5. 传输状态机

`Disconnected -> Connecting -> Connected -> Degraded -> Reconnecting -> Disconnected`。

- `Connecting` 超时：`HIL-COM-001`。
- 连续心跳丢失超过 `HeartbeatMissLimit`：进入 `Degraded`。
- 重连必须递增 `ClockEpoch`，旧代次帧全部拒收。
- `Connected` 恢复后必须执行一次 Signal State Reconciliation。

## 6. 配置 Schema

```yaml
transport:
  protocol: s7
  endpoint: 192.168.10.20:102
  connectTimeoutMs: 3000
  heartbeatIntervalMs: 500
  heartbeatMissLimit: 3
  receiveBufferSize: 4096
  maxInFlightFrames: 1024
session:
  lateInputPolicy: reject
  duplicateWindow: 4096
  reconnectBackoffMs: [100, 500, 2000, 5000]
  requireMappingHashMatch: true
```

## 7. 错误码

| 错误码 | 含义 | 恢复策略 |
|---|---|---|
| HIL-COM-001 | 连接超时 | 指数退避重连 |
| HIL-COM-002 | 握手协议不兼容 | 标记 Failed，禁止自动重试 |
| HIL-COM-003 | ClockEpoch 不匹配 | 丢弃帧并请求重新握手 |
| HIL-COM-004 | 输入重复 | 记录审计，不重复注入 |
| HIL-COM-005 | 输出 Ack 超时 | Critical 输出触发安全策略 |
| HIL-COM-006 | 映射 Hash 不一致 | 拒绝进入 Ready |

## 8. 验收标准

- 协议适配器可替换且不引入 Kernel 依赖。
- 断线重连后旧代次帧 100% 被拒绝。
- 同一 `ExternalSequence` 重复发送不会产生第二条输入事件。
- 关键输出 Ack 超时在一个安全周期内进入安全值。
- Transport、Session、IO Runtime、Kernel 的 TraceId/CorrelationId 可串联。
