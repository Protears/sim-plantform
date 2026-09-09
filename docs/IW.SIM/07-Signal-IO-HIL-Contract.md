# Part 07：Signal/IO Runtime 与 HIL 契约

## 1. 责任边界

Signal/IO Runtime 将设备能力、PLC 点位、HIL 信号统一为逻辑信号；它拥有信号类型、质量、边沿、保持时间与映射版本，不拥有 TCP/S7 等协议细节。HIL 只消费 `IoSignalContract`，不得自行推断点位语义。

## 2. 信号模型

```csharp
public sealed record IoSignalValue(
    string SignalId, IoDataType DataType, object? Value,
    SignalQuality Quality, long SimTick, long Sequence,
    string MappingHash, Guid CorrelationId);

public sealed record IoSignalContract(
    string ContractVersion, string MappingHash,
    IReadOnlyList<IoSignalDefinition> Signals);
```

`IoSignalDefinition` 必须包含：`SignalId`、`Direction`、`DataType`、`Address`、`Unit`、`IsCritical`、`FailSafeValue`、`PulseHoldTicks`、`OutputProfile`。

## 3. 校验规则

1. 同一 `Address` 不得被两个不同方向的信号占用。
2. `DataType` 与协议能力不匹配时，合同不能进入 Active。
3. 关键输出必须有 `FailSafeValue`，且必须通过类型校验。
4. 合同激活前计算 Canonical JSON 的 SHA-256 作为 `MappingHash`。
5. 运行中合同只读，变更必须创建新版本并重新握手。

## 4. 信号生命周期

`Declared -> Validated -> Activated -> Observed -> Invalidated -> Retired`。

输入的 `Invalidated` 只影响质量，不直接清零值；安全层根据 `Quality` 与 `IsCritical` 决定是否应用安全值。输出的 `Retired` 不能再产生新帧。

## 5. 事件契约

```json
{
  "eventType": "HilInputAccepted",
  "signalId": "sensor.10101.blocked",
  "externalSequence": 18422,
  "targetSimTick": 901230,
  "clockEpoch": 7,
  "mappingHash": "sha256:...",
  "quality": "Good",
  "correlationId": "..."
}
```

## 6. 测试矩阵

| 场景 | 预期 |
|---|---|
| 地址重叠 | 拒绝激活，返回 `IO-CON-001` |
| 类型不兼容 | 拒绝激活，返回 `IO-CON-002` |
| MappingHash 不一致 | Session 不得进入 Ready |
| 重复输入 | 只保留第一条记录 |
| Critical 输出失联 | 输出切换至安全值 |
| 脉冲宽度不足 | 自动补足 `PulseHoldTicks` |
