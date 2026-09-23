# Slice J：Task/MCS/WCS 编排工程实施切片

## 目标

实现最小可运行闭环：业务 Task 经过 MCS 拆分、WCS 路径与交通预留、DeviceCommand 执行、Occupancy Transfer 提交后，产生可查询、可补发、可 Replay 的 TaskCompleted 事实。

## 交付范围

1. Contracts：TaskSpec、TaskExecution、TaskEventEnvelope、TaskCompleted。
2. Application：ITaskOrchestrator、IMcsPlanner、IWcsDispatcher。
3. Data：task_instance、task_dependency、task_execution_attempt、Outbox。
4. Runtime：RouteResolver、TrafficReservation、DeviceRuntime、OccupancyAuthority 适配。
5. API：创建、查询、取消、重规划。
6. Tests：TO-001～TO-015。

## 实施顺序

- J1：Task Schema/Migration。
- J2：Dependency Graph Validator + PlanHash。
- J3：Route/Reservation 适配和幂等。
- J4：Command Dispatch 与 Transfer Correlation。
- J5：T-COMPLETE 事务与 TaskCompleted Outbox。
- J6：API/SignalR/Replay 查询。
- J7：P0 门禁和 72h 稳定性。

## 完成定义

- 可创建 MoveCargo Task 并得到稳定 PlanHash。
- Reservation 失败不产生 DeviceCommand。
- Device 完成但 Transfer 未提交时 Task 不完成。
- Transfer 提交后 TaskCompleted 可由 EventSequence 查询和 SignalR 补发。
- Replay 使用原 TaskSnapshot、LayoutSnapshot、DeviceBindingSnapshot。
- P0 测试全绿，且无孤儿 Task/Reservation/Lock。
