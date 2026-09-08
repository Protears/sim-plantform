# Part12 HIL - IO Contract 与映射契约

## 1. Contract 模型

IO Contract 是 PLC/外部控制器与 Signal/IO Runtime 之间的版本化接口，不允许通过散落字符串地址直接耦合。

```csharp
public sealed record IoSignalContract(
    string SignalId,
    string Direction,
    string DataType,
    string Address,
    string Semantic,
    string Criticality,
    string UpdateMode,
    double? Min,
    double? Max,
    int? PulseHoldTicks);
```

方向只允许 `Input`、`Output`、`Bidirectional`；Bidirectional 必须定义仲裁规则。

## 2. 校验规则

启动 Handshake 前必须校验：

1. SignalId 唯一；
2. Address 无非法重叠；
3. 数据类型与 Adapter Capability 匹配；
4. 枚举值完整；
5. Critical 信号具有 Fail-Safe 值；
6. Pulse 信号定义保持 Tick；
7. Mapping Hash 与 IntegrationProfile 一致。

Contract Hash：

```text
SHA-256(CanonicalContractJson)
```

运行期间 Contract 不允许原地修改；升级必须创建新版本并重新 Handshake。

## 3. 输入/输出处理

输入：Decode -> RangeCheck -> Deduplicate -> Record -> Schedule。

输出：RuntimeChange -> ContractPolicy -> SafetyGuard -> Encode -> Publish -> Ack。

任何 Decode/Encode 异常必须包含 SignalId、Address、SessionId、RunId，但敏感协议内容按策略脱敏。

## 4. 失败策略

| Criticality | ACK 超时 | 断线 |
|---|---|---|
| Critical | Pause/Fail | Fail-Safe + Failed |
| Important | Retry 后 Degraded | Degraded |
| BestEffort | 记录丢失 | 可恢复后继续 |

## 5. 工程产物

建议 `IW.SIM.Signal.Contracts` 保存 Contract DTO 与 Validator；`IW.SIM.HIL` 仅消费抽象接口，具体 S7/Modbus/OPC UA Adapter 位于独立基础设施程序集。
