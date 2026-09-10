# IW.SIM Part07 Signal IO Runtime 架构设计

Signal IO Runtime 是设备模型、PLC 过程映像和 HIL 信号契约之间的唯一逻辑信号边界。原有职责保持不变，但本版本补充周期提交、质量传播、版本 Hash 和 HIL 接入约束。

## 1. 核心链路

```text
Device Runtime -> Signal IO -> Input Staging -> InputCommit -> PLC Input Image
PLC Output Image -> OutputCommit -> Signal IO -> Device Runtime / HIL
```

## 2. 规范化信号模型

```csharp
public sealed record SignalValue(
    string SignalId,
    PlcDataType DataType,
    ReadOnlyMemory<byte> RawValue,
    SignalQuality Quality,
    long SourceSequence,
    long SimTick);

public interface ISignalIoRuntime
{
    void StageInput(SignalValue value);
    void StageOutput(SignalValue value);
    InputImageCommit CommitInputs(long simTick);
    OutputImageCommit CommitOutputs(long simTick);
}
```

禁止使用 `object Value` 作为跨模块契约；协议适配器负责字节到类型的转换，Signal IO 负责业务信号语义和映射校验。

## 3. 质量与安全规则

- `Good`：允许进入 PLC 输入映像。
- `Uncertain`：允许进入普通逻辑，但不得驱动 Critical Output。
- `Bad`：输入仍可审计，但默认不覆盖上一有效值；安全策略可强制置位。
- Critical Output 必须声明 `FailSafeValue`、`AckRequired`、`AckTimeoutTicks`。

## 4. 映射与版本

每个映射包含 `SignalId、PlcId、Address、Direction、DataType、ContractVersion、MappingHash`。配置加载阶段执行地址重叠、类型长度、方向和唯一性校验；失败不得启动 PLC Runtime。

## 5. 周期一致性

`StageInput -> InputCommit -> Plc Logic -> OutputCommit` 必须在 Part06 扫描周期内按固定顺序执行。HIL 输入先写入 Part10 InputRecord，再在目标 `SimTick` 的 TickBoundary 注入；HIL 输出只能消费已提交 Output Image。

## 6. 故障处理

|错误码|条件|处理|
|-|-|-|
|IO-RT-001|映射 Hash 不一致|Session 保持 Handshaking|
|IO-RT-002|地址冲突|拒绝加载配置|
|IO-RT-003|Quality=Bad 的关键输入|触发安全策略并发布事件|
|IO-RT-004|Output Commit Hash 不一致|保留上一输出并进入 Degraded|

## 7. 验收项

1. 同一 SimTick 内 Signal 更新不会影响已开始的 PLC Logic。
2. 映射修改会生成新 ContractVersion 和 MappingHash。
3. 重复 `ExternalSequence` 不得产生第二个 SignalValue。
4. 旧 ClockEpoch 信号不得覆盖当前输入映像。
5. OutputCommit 失败时设备保持上一有效命令。
6. 100 个 PLC 并发提交时不存在 SignalId/PlcId 串线。

详见：`docs/IW.SIM/07-Signal-IO-HIL-Contract.md`、`docs/IW.SIM/06-PLC-Process-Image-Design.md`。
