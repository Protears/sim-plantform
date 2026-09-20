# Part 06：设备能力与点位映射契约

## 1. 目标

把设备能力、命令点、反馈点和 PLC/Signal IO 映射拆成可校验契约，禁止控制器通过字符串和私有约定读取点位。

## 2. 契约模型

```csharp
public sealed record CapabilityDescriptor(
    string CapabilityId,
    string Version,
    IReadOnlySet<string> RequiredInputs,
    IReadOnlySet<string> RequiredOutputs,
    bool SupportsCancellation);

public sealed record PointBinding(
    string PointId,
    string SignalType,
    string Address,
    string DataType,
    string QualityPolicy);

public interface IDeviceCapabilityResolver
{
    CapabilityResolution Resolve(
        DeviceDefinitionDescriptor definition,
        DeviceConfigurationDocument configuration);
}

public interface IPointMappingValidator
{
    MappingValidationResult Validate(
        IReadOnlyCollection<PointBinding> bindings,
        CapabilityResolution capabilities);
}
```

## 3. 映射约束

- 每个 Capability 必须声明必需输入和输出。
- 同一 `PointId` 不得映射到不同语义。
- 同一 PLC Address 可以承载复合字段，但必须显式声明 Bit/Byte/Word overlay。
- `QualityPolicy=Critical` 的点位缺失或 Bad 时，相关命令不得进入 Executing。
- 输出点必须声明边沿/电平/值变化触发语义。
- 反馈点必须声明 `ProcessImageVersion` 和 `CycleSequence` 传播策略。

## 4. 映射哈希

`MappingHash` 覆盖 Capability、PointBinding、Overlay、QualityPolicy、TriggerPolicy；不包含仅用于显示的名称和颜色。

## 5. 错误码

- `POINT-MAP-404`：必需点位缺失。
- `POINT-MAP-409`：地址或语义冲突。
- `POINT-MAP-422`：数据类型/触发策略非法。
- `POINT-MAP-423`：Critical 质量策略未满足。

## 6. 测试

- 缺失一个必需输入时绑定失败。
- RisingEdge 输出在同一扫描周期只生成一次 Intent。
- Bad Quality 输入不能生成 Critical DeviceCommand。
- 映射哈希变化必须阻断旧 BindingSnapshot 复用。
