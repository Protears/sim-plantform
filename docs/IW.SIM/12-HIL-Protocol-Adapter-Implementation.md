# Part 12：HIL 协议适配器实施设计

## 1. 适配器分层

`FrameTransport` 处理连接与字节流；`ProtocolAdapter` 处理握手、编解码与协议错误；`SignalMapper` 处理地址到逻辑信号的映射；`HilSession` 处理时钟、序列和生命周期。四层之间只通过 DTO 传递数据。

## 2. C# DTO

```csharp
public sealed record HilRawFrame(
    ReadOnlyMemory<byte> Payload,
    DateTimeOffset ReceivedAt,
    long TransportSequence,
    string ClockEpoch);

public sealed record HilSignalFrame(
    Guid SessionId, string MappingHash, long ExternalSequence,
    long? TargetSimTick, DateTimeOffset? SourceTimestamp,
    IReadOnlyDictionary<string, object?> Values,
    Guid CorrelationId);
```

## 3. 解码约束

解码器不得抛出未分类异常；协议错误统一转换为 `ProtocolDecodeResult.Invalid(code, offset, reason)`。单帧最大 64KB，单字段最大 4KB，整数溢出、字符串非法编码、长度字段越界均拒绝。

## 4. 性能与背压

接收队列容量默认 1024；达到 80% 记录告警，达到 100% 对非关键输入丢弃并写入计数器，关键输入触发 `HIL-TRN-004` 并进入 Degraded。编解码线程不得阻塞 Session 状态线程。

## 5. 可测试性

适配器必须支持内存流注入、确定性时间源和故障帧生成器。最低测试包括：握手成功/失败、半包重组、粘包拆分、无效长度、重复序列、异常断线、重连后旧 Epoch 帧、编码往返一致性。
