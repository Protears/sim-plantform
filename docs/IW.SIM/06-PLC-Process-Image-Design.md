# PLC 过程映像与内存一致性设计

## 1. 设计目标

过程映像是 PLC Runtime 与 Signal/IO Runtime 之间的唯一周期性数据边界。它解决现场信号在逻辑执行过程中变化导致的非确定性，并将输入采集、程序执行、输出刷新拆成可审计阶段。

## 2. 内存分区

```text
Input Image  : I 区、传感器、HIL 输入
Output Image : Q 区、设备命令、HIL 输出
Marker       : M 区、内部变量
Data Block   : DB 区、结构化业务数据
Diagnostics  : 诊断、质量、更新时间戳
```

所有值必须携带 `DataType、Quality、SourceSequence、UpdatedAtSimTick`。禁止以 `object` 作为跨模块契约类型。

## 3. 结构化模型

```csharp
public readonly record struct PlcAddress(
    PlcArea Area, int ByteOffset, int BitOffset, int Length);

public sealed record ProcessImageCell(
    PlcAddress Address,
    PlcDataType DataType,
    ReadOnlyMemory<byte> RawValue,
    SignalQuality Quality,
    long SourceSequence,
    long UpdatedAtSimTick);

public interface IProcessImage
{
    ProcessImageVersion Version { get; }
    bool TryRead(PlcAddress address, out ProcessImageCell cell);
    void StageInput(ProcessImageCell cell);
    void StageOutput(ProcessImageCell cell);
    ProcessImageSnapshot CommitInput(long simTick);
    ProcessImageSnapshot CommitOutput(long simTick);
}
```

## 4. 一致性规则

|阶段|允许操作|禁止操作|
|-|-|-|
|InputCollect|写入 StagingInput|修改 ActiveInput|
|InputCommit|原子切换输入版本|逻辑读取 StagingInput|
|LogicExecute|读取 ActiveInput、写入 StagingOutput|直接访问设备或 DB|
|OutputCommit|原子切换输出版本|回写逻辑变量|
|Publish|发布已提交输出|发布未提交缓存|

## 5. 地址与类型校验

- 地址范围必须在 PLC Profile 定义内。
- 同一地址的 Bit/Byte/Word 重叠必须显式声明为同一复合字段，否则拒绝加载。
- `BOOL` 只能使用有效 BitOffset；数值类型必须满足字节对齐和长度约束。
- 输入映射与输出映射方向冲突时返回 `PLC-MEM-001`。
- 未知 Quality 的输入不得驱动 Critical Output。

## 6. 版本与快照

`ProcessImageVersion` 由 `RunId + PlcId + SimTick + CycleSequence + ImageKind` 组成。每次 Commit 生成 Hash；Part10 Snapshot 必须保存最近一次 Input/Output Image Version 与 Hash，恢复后先校验再允许继续扫描。

## 7. 异常处理

|错误码|条件|处理|
|-|-|-|
|PLC-MEM-001|地址冲突或方向冲突|拒绝配置加载|
|PLC-MEM-002|类型/长度不匹配|拒绝本次写入，保留旧值|
|PLC-MEM-003|版本过期|重新读取当前版本，禁止覆盖|
|PLC-MEM-004|提交 Hash 不一致|Session 进入 Degraded，触发诊断事件|

## 8. 测试矩阵

1. 同一 SimTick 内输入变化不得影响已开始的 LogicExecute。
2. OutputCommit 失败时设备保持上一个有效输出。
3. 断线输入 Quality=Bad 时 Critical Output 进入安全值。
4. 重叠地址检测覆盖 Bit/Byte/Word/DB 四种组合。
5. Snapshot Restore 后 Image Hash 与 Event Sequence 一致。
6. 100 个 PLC 并发 Commit 时无跨实例数据污染。
