# Part 05：设备配置 Schema 与确定性校验

## 1. 配置分层

```yaml
schemaVersion: 1
instanceId: conveyor-001
definition:
  id: conveyor.standard
  version: 1.2.0
geometry:
  lengthMm: 1200
  widthMm: 800
runtime:
  mode: auto
  tickPhase: DeviceExecute
  maxCommandQueue: 32
capabilities:
  - CargoTransfer
  - SensorFeedback
bindings:
  plc:
    plcId: plc-01
    inputMap: photoeye.entry
    outputMap: motor.forward
  resources:
    - track:zone-a
```

## 2. 校验规则

- `schemaVersion` 必须在运行时支持范围内。
- `instanceId` 在 RunProfile 内唯一。
- 几何尺寸必须大于零，单位固定为 mm。
- `tickPhase` 只能取 Kernel 已注册 Phase。
- `maxCommandQueue` 必须在设备类型允许范围内。
- Capability 必须存在于 Definition 的能力集合。
- PLC PointId、ResourceId、SensorId 不得重复绑定。
- 未声明的额外字段按配置策略拒绝或进入警告清单，不得静默忽略。

## 3. 确定性摘要

`DeviceConfigHash = SHA-256(CanonicalJson(DeviceDefinition + Geometry + Runtime + Capabilities + Bindings))`。

CanonicalJson 要求：属性名按 ordinal 排序、数组按语义顺序、数字使用固定格式、空值显式保留。

## 4. 生命周期

`Draft → Validated → Approved → BoundToRun → Frozen → Retired`

进入 `Frozen` 后，几何、能力、Phase、Mapping 和控制器配置不得热更新；非确定性观测参数必须显式标记为 `OperationalOnly`。

## 5. 结构化产物

- `DeviceConfigurationDocument`
- `DeviceConfigurationValidator`
- `DeviceConfigHashCalculator`
- `DeviceConfigRejected` Event

## 6. 验收

- 相同规范化配置在不同节点计算出相同 Hash。
- 配置字段顺序变化不得改变 Hash。
- Frozen 配置变更必须拒绝并记录 `CONFIG-FROZEN-409`。
