# Layout 与 Device Binding 契约

## 1. 绑定关系

`LayoutSnapshot → DeviceBindingSnapshot → DeviceRuntime/PLC Mapper/SignalIO`。
设备绑定不得自行创建 Place、Track 或 Point；所有空间引用必须来自冻结布局快照。

## 2. 结构化模型

```csharp
public sealed record LayoutDeviceBinding(
    Guid DeviceBindingId,
    Guid DeviceInstanceId,
    Guid LayoutSnapshotId,
    Guid PrimaryTrackId,
    IReadOnlyList<Guid> ControlledPlaceIds,
    IReadOnlyList<Guid> PointIds,
    string MappingHash);

public interface ILayoutDeviceBinder
{
    BindingResult Bind(LayoutSnapshot layout, DeviceConfigurationDocument config);
}
```

## 3. 规则

- DeviceInstance 必须拥有唯一 `PrimaryTrackId`，多轨设备显式声明 `TrackBindings`。
- ControlledPlace 必须属于设备能力声明；普通输送机不得绑定不支持的 Place 类型。
- PointId、PlaceId、TrackId 不允许跨 LayoutSnapshot 引用。
- BindingHash = SHA-256(LayoutHash + DeviceConfigHash + MappingHash + CapabilitySetVersion)。
- 绑定快照生成后，Runtime、PLC、Recovery、Replay 只能读取同一 `DeviceBindingSnapshotId`。

## 4. 失败语义

- `LAYOUT-BIND-404`：空间实体不存在。
- `LAYOUT-BIND-409`：设备与空间资源冲突。
- `LAYOUT-BIND-422`：能力与空间类型不兼容。
- `LAYOUT-BIND-423`：空间已被其他设备独占。

## 5. 完整闭环

`Layout Validate → Device Bind → BindingHash → Run Bind → Freeze → Runtime Consume → Recovery/Replay Reuse`。
