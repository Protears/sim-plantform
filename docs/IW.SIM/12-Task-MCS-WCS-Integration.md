# Part 12：Task / MCS / WCS 编排与执行集成

## 1. 章节定位

本章定义业务任务如何进入仿真运行时，并与 LayoutSnapshot、DeviceBindingSnapshot、Route、Traffic Reservation、DeviceCommand、Occupancy Transfer、EventSequence、Recovery/Replay 建立一致的执行链路。

## 2. 核心原则

1. Task 表达业务目标，不表达瞬时设备状态。
2. MCS 负责拆分、依赖和优先级；WCS 负责可执行调度；Device Runtime 负责设备动作。
3. OccupancyAuthority 是货载位置事实权威。
4. TaskCompleted 只能由 Occupancy Commit 授权。
5. Task、Route、Reservation、Command、Transfer 必须共享 RunId、CorrelationId 和冻结快照引用。

## 3. 端到端边界

```text
Task API / WCS
  → MCS Planner
  → Route Resolver
  → Traffic Reservation
  → DeviceCommand Admission
  → Device Runtime
  → Occupancy Transfer
  → TaskCompleted Event / Outbox
```

## 4. 快照引用

TaskExecution 必须持有：

- `LayoutSnapshotId/Hash`
- `DeviceBindingSnapshotId/BindingHash`
- `RunProfileId/SchemaVersion`
- `WorldVersion`
- `PlanHash`

任何快照不一致都必须要求 Replan，不得继续 Dispatch。

## 5. 恢复与回放

Recovery 时按 Task → Plan → Reservation → Command → Transfer 的事实游标恢复；Replay 使用原始 TaskSnapshot 和空间/设备绑定快照，不读取当前最新配置。

## 6. 与 Part14 的工程门禁

Task 编排属于 Slice J。P0 门禁包括：依赖环检测、WorldVersion 冲突、Reservation 竞争、Transfer 前完成事件阻断、TaskCompleted 事务一致性、SignalR 补发和 Replay Hash 一致性。
