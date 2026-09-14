# Part 06：PLC 输出到设备命令的映射契约

## 1. 目标

定义 PLC Process Image、Signal/IO Runtime 与 Device Runtime 之间的唯一映射路径，解决“PLC 已输出但设备命令未生成”“同一输出重复触发”“设备状态回写覆盖输入”等问题。

## 2. 映射模型

```csharp
public sealed record PlcCommandBinding(
    Guid BindingId,
    Guid PlcId,
    Guid DeviceId,
    string TriggerSignal,
    string CommandType,
    string PayloadSchema,
    int EdgeHoldTicks,
    int Priority,
    bool RequireAck);

public interface IPlcDeviceCommandMapper
{
    IReadOnlyList<DeviceCommandIntent> Evaluate(
        ProcessImageSnapshot image,
        PlcCycleContext context);
}
```

触发器支持 RisingEdge、FallingEdge、Level、ValueChanged 四类。Mapper 只产生不可变 `DeviceCommandIntent`，不直接写设备状态或数据库。

## 3. 去重与边沿语义

- Rising/Falling Edge 以 ProcessImageVersion 为基准，只允许每次边沿生成一次 Intent。
- Level 信号必须配合 `cooldown_ticks`，避免每个扫描周期重复下发。
- ValueChanged 必须比较上次已提交值，不比较尚未提交的临时 Buffer。
- Intent 必须携带 `cycle_sequence`、`source_signal_id`、`request_hash`，后续由 Command Admission 生成或复用 CommandId。

## 4. 反馈映射

Device Runtime 输出的运行事实按以下顺序写回：

`Device State → SignalIo Output Source → PLC Input Image Commit`。

禁止 Device Runtime 直接修改 PLC Input Buffer。对于 Cargo 到位、设备故障、Transfer Commit 等信号，必须定义 Quality 和来源事件；`Bad` 状态不得被映射为正常到位。

## 5. 时序与异常

正常路径：`InputCommit → PLC Logic → OutputCommit → Mapper → Admission → DeviceRuntime`。若 Mapper 生成多个 Intent，按 `Priority → DeviceId → BindingId` 固定排序。设备忙、版本冲突、资源占用分别映射为 Busy/Conflict/Fault 信号，不得吞掉异常。

## 6. 验收项

1. 同一扫描周期同一 RisingEdge 只产生一个 Intent。
2. 断线重连后旧 CycleSequence 不得重复触发。
3. RequestHash 相同的 Intent 重试复用原 CommandId。
4. Device Runtime 故障能在下一周期通过 Quality=Bad 回写 PLC。
5. Transfer Commit 未完成时不得回写“目标到位完成”。
6. 映射配置缺少 PayloadSchema 时启动校验失败。
