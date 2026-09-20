# Part 05：设备定义、实例绑定与运行时装配基线

## 1. 目标

将设备类型模板、设备实例、能力、信号映射、控制器和运行态装配收敛为单一契约，避免 Device Runtime 直接从 UI/数据库拼装设备。

## 2. 核心实体

- `DeviceDefinitionId`：设备类型定义，例如 `conveyor.standard.v1`。
- `DeviceInstanceId`：场景中的设备实例。
- `DeviceBindingId`：实例与 PLC/Signal/Communication 资源的绑定。
- `CapabilitySetVersion`：能力集合版本。
- `MappingHash`：I/O、命令点位和反馈点位的稳定摘要。
- `ControllerProfileId`：设备控制器实现配置。

## 3. 装配关系

```text
DeviceDefinition
  ├─ CapabilitySet
  ├─ StateModel
  ├─ CommandSchema
  ├─ SignalSchema
  └─ ControllerProfile
          ↓ bind
DeviceInstance
  ├─ Geometry
  ├─ RuntimeOptions
  ├─ PlcBinding
  └─ CommunicationBinding
```

## 4. C# 接口

```csharp
public interface IDeviceDefinitionRegistry
{
    ValueTask<DeviceDefinitionDescriptor?> GetAsync(
        string definitionId, string version, CancellationToken ct);
}

public interface IDeviceInstanceBinder
{
    ValueTask<DeviceBindingResult> BindAsync(
        DeviceInstance instance,
        DeviceDefinitionDescriptor definition,
        BindingContext context,
        CancellationToken ct);
}

public interface IDeviceRuntimeFactory
{
    ValueTask<IDeviceRuntime> CreateAsync(
        DeviceBinding binding, CancellationToken ct);
}
```

## 5. 绑定前置条件

1. Definition、CapabilitySet、CommandSchema、SignalSchema 版本必须兼容。
2. DeviceInstanceId、PointId、ResourceId 在 Run 内唯一。
3. MappingHash 必须由规范化 JSON 计算，禁止使用字段顺序不稳定的摘要。
4. ControllerProfile 必须与设备类型和运行模式匹配。
5. 绑定成功后生成不可变 `DeviceBindingSnapshot`，运行中禁止变更确定性字段。

## 6. 错误码

- `DEVICE-DEF-404`：设备定义不存在。
- `DEVICE-DEF-409`：定义版本不兼容。
- `DEVICE-BIND-409`：实例绑定冲突。
- `DEVICE-BIND-422`：点位/资源配置非法。
- `DEVICE-BIND-503`：控制器装配失败。

## 7. 验收

- 同一 Definition + Instance + MappingHash 重复装配必须幂等。
- MappingHash 变化时必须生成新的 BindingSnapshot，旧 Run 不得静默复用。
- 未通过绑定校验的设备不得进入 `Ready`。
