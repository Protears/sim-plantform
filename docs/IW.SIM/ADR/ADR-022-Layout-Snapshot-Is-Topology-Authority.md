# ADR-022：LayoutSnapshot 是拓扑权威

## 状态
Accepted

## 背景
设备绑定、Occupancy、Route、Traffic、PLC PointMapping 如果读取不同版本的布局，会产生空间事实分叉和不可重放结果。

## 决策
1. LayoutSnapshot 是 Run 的唯一空间拓扑权威。
2. Run 绑定后生成 `LayoutSnapshotId`、`LayoutHash`，并进入 Frozen。
3. DeviceBindingSnapshot 必须引用 LayoutSnapshotId。
4. Recovery/Replay 必须使用原始 LayoutSnapshot，禁止读取当前最新布局。
5. Route、Traffic Reservation、Occupancy Transfer 均必须携带 LayoutSnapshotId。

## 后果
- 支持确定性 Replay 和审计追踪。
- 布局变更必须生成新版本，不能热更新既有 Run。
- 需要在数据层增加 snapshot 外键和 Hash 校验。
