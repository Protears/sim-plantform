# Layout/Topology 验证矩阵

| ID | 场景 | 预期 |
|---|---|---|
| LT-001 | 重复 PlaceId | 阻断验证 |
| LT-002 | 跨快照 Link | 阻断验证 |
| LT-003 | 隐式反向边 | 阻断验证 |
| LT-004 | 孤立 Place | Warning 或阻断，按 LayoutPolicy |
| LT-005 | 设备绑定不存在 Track | `LAYOUT-BIND-404` |
| LT-006 | 设备能力与 Place 类型不兼容 | `LAYOUT-BIND-422` |
| LT-007 | 同一 Track 被独占设备重复绑定 | `LAYOUT-BIND-423` |
| LT-008 | 同输入重复路径解析 | RouteHash 一致 |
| LT-009 | Route 资源锁顺序错误 | 阻断执行 |
| LT-010 | Reservation 失败 | 不得进入 Executing |
| LT-011 | Recovery 使用旧 LayoutSnapshot | 通过且 Hash 一致 |
| LT-012 | Replay 使用当前最新布局 | 必须失败 |

## 门禁

P0：LT-001/002/003/005/006/007/009/010/012 全部通过；P0 失败不得进入 PLC/HIL 联调。P1：LT-004/008/011。
