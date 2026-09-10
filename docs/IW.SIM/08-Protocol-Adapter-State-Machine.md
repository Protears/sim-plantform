# 工业协议适配器状态机与故障模型

## 1. 适配器职责

协议适配器负责连接参数、字节流编解码、帧完整性和协议级 Ack；不负责 PLC 变量语义、设备运动或仿真时间推进。业务数据通过 `HilSignalFrame` / `PlcProcessImage` 交给上层。

## 2. 状态机

```text
Created -> Connecting -> Handshaking -> Ready
    |          |             |            |
    v          v             v            v
 Disposed   Degraded       Failed      Reconnecting
                                  ^         |
                                  +---------+
```

- `Connecting`：TCP/串口/WebSocket 连接尚未完成。
- `Handshaking`：版本、能力、MappingHash、ClockEpoch 协商。
- `Ready`：允许收发业务帧。
- `Degraded`：连接仍存在，但质量、RTT、Ack 或解析错误超过阈值。
- `Reconnecting`：仅允许发送握手/心跳，不允许旧 Epoch 业务帧通过。

## 3. 接口契约

```csharp
public interface IProtocolAdapter : IAsyncDisposable
{
    ProtocolId Protocol { get; }
    AdapterState State { get; }
    ValueTask<HandshakeResult> HandshakeAsync(
        HandshakeRequest request, CancellationToken ct);
    ValueTask<DecodeResult> DecodeAsync(
        ReadOnlyMemory<byte> buffer, CancellationToken ct);
    ValueTask<EncodeResult> EncodeAsync(
        HilSignalFrame frame, CancellationToken ct);
    ValueTask OnConnectionLostAsync(ConnectionFault fault, CancellationToken ct);
}
```

## 4. 帧处理规则

1. Transport 层先完成半包/粘包重组并附加 `TransportSequence`。
2. Adapter 校验长度、校验和、功能码与协议版本。
3. 非法帧只产生 `ProtocolFrameRejected`，不得进入 InputRecord。
4. 解析成功后生成 `ExternalSequence`；若外部序列不存在，适配器分配单调递增本地序列。
5. 任何重连都必须递增 `ClockEpoch`，旧 Epoch 帧一律拒绝。

## 5. 故障码

|错误码|含义|恢复动作|
|-|-|-|
|COM-ADP-001|握手版本不兼容|进入 Failed，禁止自动重试|
|COM-ADP-002|MappingHash 不一致|保持 Handshaking，等待配置修复|
|COM-ADP-003|帧校验失败|丢弃帧并计数，超过阈值进入 Degraded|
|COM-ADP-004|协议解析异常|隔离单帧，不关闭连接；连续超阈值重连|
|COM-ADP-005|ClockEpoch 过期|拒绝帧，记录诊断|
|COM-ADP-006|Ack 超时|交由 HIL SafetyGuard 触发安全策略|

## 6. 并发与背压

每个 Adapter 具有独立接收队列和发送队列，容量由 Profile 配置。接收队列满时：Critical 帧触发 `COM-ADP-007` 并进入 Degraded；非关键遥测帧允许丢弃。适配器不得使用无界 Channel，不得在协议线程中同步等待数据库。

## 7. 观测字段

每次帧处理记录 `SessionId、Protocol、TransportSequence、ExternalSequence、ClockEpoch、FrameType、PayloadLength、DecodeLatencyTicks、CorrelationId`。指标：`protocol_decode_error_total`、`adapter_reconnect_total`、`adapter_queue_depth`、`adapter_ack_timeout_total`。

## 8. 验收项

- 连续发送 1 帧拆分为 3 个 TCP 包，必须只生成 1 个业务帧。
- 3 帧合并为 1 个 TCP 包，必须生成 3 个业务帧。
- 重连后旧 Epoch 帧不得改变 Process Image。
- MappingHash 不一致时不能进入 Ready。
- 队列满载时 Critical/Telemetry 的处理策略符合配置。
