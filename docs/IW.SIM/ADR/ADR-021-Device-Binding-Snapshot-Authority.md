# ADR-021：Device BindingSnapshot 作为运行时装配权威

## 状态

Accepted

## 背景

设备定义、实例配置、PLC 点位、Capability 和 ControllerProfile 由多个文档和运行时组件描述。若运行时直接从可变配置读取，容易造成 MappingHash、ProcessImageVersion、Replay 和 Recovery 不一致。

## 决策

1. `DeviceBindingSnapshot` 是单个 Run 中设备运行时装配的唯一权威。
2. Snapshot 由 Definition + Configuration + Capability + PointMapping + ControllerProfile 规范化计算生成。
3. Run 进入 `BoundToRun` 后 Snapshot 不可变；运行期间只能产生新的 Profile/Run，不得修改当前 Snapshot。
4. Device Runtime、PLC Mapper、Signal IO、Recovery、Replay 必须读取同一个 SnapshotId。
5. SnapshotId、MappingHash、CapabilitySetVersion 必须写入 Run 初始化事实和每个关键设备事件。

## 后果

- 优点：装配可复现、配置变更可审计、Replay/Recovery 可验证。
- 代价：配置修改必须重新生成 BindingSnapshot，不能直接热更新。
- 迁移：旧文档中直接读取 DeviceConfig 的接口必须改为读取 BindingSnapshot。

## 验收

- 同一 SnapshotId 下不同节点计算出的 MappingHash 相同。
- Snapshot 不一致时 Run 不得进入 Ready。
- Replay 使用原 SnapshotId，禁止读取当前最新配置。
