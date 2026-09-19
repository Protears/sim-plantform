# Part 14：Run Profile 与运行配置 Schema

## 1. 目标

本文件把 RunProfile、设备实例、PLC 映射、通信/HIL、调度和数据策略收敛为可校验、可版本化、可迁移的配置契约。配置是运行时装配的输入，不是运行态事实；运行开始后不得原地修改影响确定性结果的字段。

## 2. 配置分层

| 层级 | 责任 | 可变性 |
|---|---|---|
| PlatformProfile | 版本、时钟、存储、观测 | 启动前 |
| RunProfile | 仿真规模、设备、PLC、HIL、策略 | Run 创建前 |
| RuntimeOverrides | 仅允许诊断和限流参数 | 受控热更新 |
| RuntimeFacts | 运行态事实 | 由运行时产生，不可由配置覆盖 |

## 3. C# Contract

```csharp
public sealed record RunProfileDocument(
    int SchemaVersion,
    Guid ProfileId,
    string Name,
    string ProductVersion,
    KernelProfile Kernel,
    IReadOnlyList<DeviceProfile> Devices,
    IReadOnlyList<PlcProfile> Plcs,
    CommunicationProfile Communication,
    DataProfile Data,
    ObservabilityProfile Observability);

public sealed record KernelProfile(
    int TickPeriodMs,
    int MaxCatchUpTicks,
    int InputQueueCapacity,
    int CriticalQueueCapacity,
    bool DeterministicMode,
    string OrderingPolicy);

public sealed record DeviceProfile(
    Guid DeviceId,
    string DeviceType,
    string ModelVersion,
    string CapabilityProfile,
    string PlaceGroupId,
    IReadOnlyDictionary<string, string> Parameters);

public sealed record PlcProfile(
    Guid PlcId,
    string Protocol,
    string EndpointProfile,
    string MappingHash,
    string ScanProfile);
```

## 4. YAML 示例

```yaml
schemaVersion: 2
profileId: 3dfc2f50-2f89-4db3-9e11-2b854cd13d01
name: asrs-line-a
productVersion: 0.1.0
kernel:
  tickPeriodMs: 30
  maxCatchUpTicks: 2
  inputQueueCapacity: 4096
  criticalQueueCapacity: 512
  deterministicMode: true
  orderingPolicy: SimTick.Phase.Priority.SourceId.LocalSequence
devices:
  - deviceId: 9b3c1c9d-5db8-4fbe-a722-4ddf3f8dbb6f
    deviceType: Conveyor
    modelVersion: conveyor.v2
    capabilityProfile: conveyor.standard
    placeGroupId: line-a
    parameters:
      speedMmPerSec: "800"
      cargoCapacity: "1"
plcs:
  - plcId: 5d1a3df0-1d20-427f-a2d4-ef6f4c4aa17a
    protocol: S7
    endpointProfile: plc-sim-01
    mappingHash: sha256:...
    scanProfile: s7-30ms
communication:
  mode: VirtualOnly
  hilEnabled: false
data:
  schemaVersion: 2
  migrationPolicy: RequireCompatible
observability:
  traceEnabled: true
  eventAuditEnabled: true
```

## 5. 校验规则

1. `SchemaVersion` 必须是运行时支持的值；不支持时拒绝启动。
2. `ProfileId`、`DeviceId`、`PlcId` 在同一配置文档内必须唯一。
3. `MappingHash` 与 PLC 映射文档、HIL Session 握手值一致后才能进入 Ready。
4. `TickPeriodMs` 必须是 1～1000 的整数；Slice D 默认 30。
5. `DeterministicMode=true` 时禁止使用随机无种子、线程完成顺序或 wall-clock 参与业务结果。
6. DeviceType、CapabilityProfile、Protocol 必须存在于已注册清单。
7. 运行开始后禁止改变 DeviceId、PlcId、MappingHash、TickPeriodMs 和 OrderingPolicy。
8. RuntimeOverrides 只能改变限流、日志级别和观测采样，不得改变领域语义。

## 6. 错误码

| Code | 触发条件 | 处理 |
|---|---|---|
| CFG-001 | SchemaVersion 不支持 | 拒绝启动 |
| CFG-002 | ID 重复 | 拒绝启动 |
| CFG-003 | MappingHash 不匹配 | Session 不进入 Ready |
| CFG-004 | 设备能力未注册 | 拒绝装配 |
| CFG-005 | 关键字段在运行中被修改 | 拒绝热更新并记录审计 |
| CFG-006 | OrderingPolicy 非法 | 拒绝启动 |

## 7. 生命周期

`Draft → Validated → Approved → BoundToRun → Frozen → Retired`

只有 `Validated` 和 `Approved` 的配置允许创建 Run。`Frozen` 配置只能用于审计、恢复和 Replay。

## 8. 直接工程任务

- 实现 JSON/YAML 反序列化和 SchemaVersion 校验。
- 实现 ProfileValidator 与注册清单。
- 在组合根装配前校验 MappingHash、能力、协议和迁移兼容性。
- 为配置冻结、热更新拒绝和审计生成 ContractTests。
