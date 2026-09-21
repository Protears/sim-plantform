# Slice H：Layout/Topology 工程实施切片

## 1. 目标

将 Layout Snapshot、设备绑定、Route、Traffic Reservation 接入现有 DeviceBindingSnapshot、Occupancy、Recovery、Replay 基线。

## 2. 模块边界

```text
Layout Authoring/Import
    ↓
Topology Validator
    ↓
LayoutSnapshot Store
    ↓
Device Binder / Route Resolver
    ↓
DeviceRuntime / Occupancy / Traffic Lock
    ↓
EventStore / Recovery / Replay
```

禁止 DeviceRuntime、PLC Mapper、Occupancy 直接解析 CAD/JSON 原始布局。

## 3. 工程任务

1. 创建 `Layout`、`Topology`、`Route` Contracts。
2. 实现 Layout Canonical JSON 与 LayoutHash。
3. 实现 `ITopologyValidator` 和 LT-001～LT-012。
4. 实现 `ILayoutDeviceBinder`，生成 BindingSnapshot。
5. 实现 `IRouteResolver` 和固定排序。
6. 实现 Traffic Reservation 与 ResourceLock 联动。
7. 将 LayoutSnapshotId/Hash 写入 Run 初始化事实、Transfer、Route、Snapshot。
8. 将 Slice H 纳入 Operational Readiness。

## 4. DoD

- 新布局可以 Draft→Frozen→BoundToRun。
- Device Runtime、PLC Mapper、Recovery、Replay 使用同一 LayoutSnapshot。
- RouteHash 可重现，Reservation 冲突可解释，Recovery 后 Hash 一致。
- P0 测试全部通过后才允许进入 PLC/HIL 联调。
