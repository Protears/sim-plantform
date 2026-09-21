# Layout、拓扑与 Place 模型

## 1. 目标

定义 Layout-first 的几何与逻辑拓扑权威，解决设备绑定、Occupancy、Transfer、Traffic 与 PLC 点位之间缺少稳定空间标识的问题。

## 2. 核心实体

| 实体 | 职责 | 稳定标识 |
|---|---|---|
| `Layout` | 一个可版本化的仓库/产线布局 | `LayoutId` |
| `Region` | 区域边界与安全域 | `RegionId` |
| `Track` | 设备可运行的连续或离散轨道 | `TrackId` |
| `Place` | 货载可占用的逻辑位置 | `PlaceId` |
| `Point` | 设备控制点、传感器或工艺点 | `PointId` |
| `Link` | Place/Track/Point 之间的有向关系 | `LinkId` |

`PlaceId` 是 Occupancy 的空间主键；`TrackPosition` 只描述 Track 内位置，不替代 `PlaceId`。

## 3. 拓扑不变量

1. `Link.FromId` 与 `Link.ToId` 必须属于同一 `LayoutVersion`。
2. 有向链路不得存在隐式反向边；反向运行必须显式建边。
3. `Place` 只能被一个活动 Occupancy Claim 占用。
4. `Point` 的语义由绑定快照声明，不能由名称推断。
5. `Track` 的长度、方向和坐标系冻结后不得在 Run 中修改。
6. `Region` 可作为 ResourceLock 的安全域资源。

## 4. 版本与冻结

`Draft → Validated → Approved → BoundToRun → Frozen → Retired`。

`LayoutHash` 由规范化 JSON 计算，字段顺序和文档空白不得影响 Hash。Run 只消费 `LayoutSnapshotId`，Recovery/Replay 必须复用原快照。

## 5. C# 契约

```csharp
public interface ILayoutSnapshotProvider
{
    ValueTask<LayoutSnapshot> GetAsync(LayoutSnapshotId snapshotId, CancellationToken ct);
}

public interface ITopologyValidator
{
    TopologyValidationResult Validate(LayoutSnapshot snapshot);
}

public sealed record LayoutSnapshot(
    Guid LayoutSnapshotId,
    Guid LayoutId,
    int Version,
    string LayoutHash,
    IReadOnlyList<RegionNode> Regions,
    IReadOnlyList<TrackNode> Tracks,
    IReadOnlyList<PlaceNode> Places,
    IReadOnlyList<PointNode> Points,
    IReadOnlyList<TopologyLink> Links);
```

## 6. 验收

- 相同布局规范化输入必须生成相同 `LayoutHash`。
- 孤立 Place、反向隐式边、重复 PointId、跨版本 Link 必须阻断绑定。
- Occupancy Transfer 只能使用 Frozen LayoutSnapshot。
