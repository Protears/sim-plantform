# IW.SIM Part08 通信架构设计

通信 Runtime 负责通道、协议、连接状态和消息传输；业务信号语义由 Signal IO Runtime 管理，PLC 扫描由 PLC Runtime 管理，仿真时间由 Simulation Kernel 管理。

## 1. 三层模型

```text
Communication Runtime
  -> Channel Layer: byte stream / connection
  -> Protocol Adapter: framing / codec / ack
  -> Integration Port: HilSignalFrame / WCS message
```

协议适配器不得直接依赖 World Model、Device Runtime、EF Core 或 Simulation Kernel 内部实现。

## 2. 规范化接口

```csharp
public interface ICommunicationChannel : IAsyncDisposable
{
    ConnectionState State { get; }
    ValueTask OpenAsync(CancellationToken ct);
    ValueTask SendAsync(ReadOnlyMemory<byte> data, CancellationToken ct);
    IAsyncEnumerable<ReadOnlyMemory<byte>> ReadFramesAsync(CancellationToken ct);
}

public interface IProtocolAdapter
{
    ProtocolId Protocol { get; }
    ValueTask<HandshakeResult> HandshakeAsync(HandshakeRequest request, CancellationToken ct);
    ValueTask<DecodeResult> DecodeAsync(ReadOnlyMemory<byte> frame, CancellationToken ct);
    ValueTask<EncodeResult> EncodeAsync(HilSignalFrame message, CancellationToken ct);
}
```

## 3. 连接和故障状态

`Created -> Connecting -> Handshaking -> Ready -> Degraded -> Reconnecting -> Failed/Disposed`。重连必须创建新的 `ClockEpoch`；旧 Epoch 的业务帧不得路由到 Signal IO。

## 4. 消息处理管线

```text
Receive bytes
 -> frame reassembly
 -> protocol checksum/version validation
 -> decode
 -> attach TransportSequence/ExternalSequence/ClockEpoch
 -> HIL Session validation
 -> Part10 InputRecord
```

发送管线：

```text
OutputCommit
 -> SafetyGuard
 -> protocol encode
 -> channel send
 -> Ack tracker
 -> timeout/fault event
```

## 5. 背压规则

每个 Channel 的接收和发送队列必须有界。遥测/非关键帧可丢弃并计数；Critical 输入或安全输出不得静默丢弃，必须触发 `COM-ADP-007` 或进入安全态。协议线程禁止同步等待数据库或 Kernel 主循环。

## 6. 与 Part06/Part07/Part12 的一致性修复

- Part06 仅消费 Process Image，不读取原始协议帧。
- Part07 仅消费规范化 SignalValue，不承担 TCP 重连。
- Part12 HIL Session 负责 Epoch、契约版本和对账，不重新实现协议编解码。
- Part10 负责 InputRecord/Event/Replay 事实链，Communication 不得直接写 World State。

## 7. 验收项

1. 半包/粘包重组后业务帧数量正确且无重复。
2. MappingHash 不一致时 Session 不能进入 Ready。
3. 重连后旧 Epoch 帧全部拒收。
4. 协议错误只隔离错误帧，达到阈值才重连。
5. Critical 输出 Ack 超时可观测并触发安全策略。
6. 网络延迟变化不改变 Replay 结果 Hash。

详见：`docs/IW.SIM/08-Protocol-Adapter-State-Machine.md`、`docs/IW.SIM/ADR/ADR-011-PLC-HIL-Time-Boundary.md`。
