# 拓扑路径解析与交通资源契约

## 1. 目标

把 Layout 的有向 Link 转换为可执行 Route，并与 Track/Region/SafetyZone 资源锁衔接。

## 2. 接口

```csharp
public interface IRouteResolver
{
    RouteResolutionResult Resolve(
        LayoutSnapshot layout,
        Guid fromPlaceId,
        Guid toPlaceId,
        RoutePolicy policy);
}

public interface ITrafficReservationService
{
    ValueTask<TrafficReservationResult> ReserveAsync(
        Route route,
        Guid runId,
        Guid transferId,
        CancellationToken ct);
}
```

## 3. 路径规则

- 只允许沿显式有向 Link 搜索。
- 结果必须包含 Place、Track、Region、SafetyZone 资源集合。
- 稳定排序：`Cost → HopCount → RouteId`；禁止依赖字典枚举顺序。
- Route 必须携带 `LayoutSnapshotId` 和 `LayoutHash`。
- Traffic Reservation 的锁顺序：`SafetyZone → Region → Track → Place`。

## 4. 状态机

`Requested → Resolving → Reserved → Executing → Released`；冲突进入 `Rejected`，超时进入 `Expired`。

## 5. 错误码

`ROUTE-404`、`ROUTE-409`、`TRAFFIC-423`、`TRAFFIC-408`、`TRAFFIC-422`。

## 6. 可验证项

- 同一 TransferId 重试返回同一 RouteId。
- 相同输入与相同 LayoutSnapshot 必须得到相同 RouteHash。
- Route 中任何实体不属于当前快照时必须拒绝。
- Reservation 未提交不得进入 Device Executing。
