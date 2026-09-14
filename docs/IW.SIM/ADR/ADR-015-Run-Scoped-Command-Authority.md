# ADR-015：Run 级命令权威与状态封存规则

## 状态

Accepted

## 背景

命令可能来自 REST、WCS 仿真、PLC Mapper、HIL 或内部恢复流程。若没有统一的 Run 级准入权威，会产生跨来源重复命令、Sealed Run 被写入、Replay 污染原运行等问题。

## 决策

1. 所有设备命令必须经过 `ICommandAdmissionService`，无论来源是什么。
2. `(RunId, CommandId)` 是唯一业务幂等键；来源只作为 `origin` 元数据，不参与生成新的业务身份。
3. Run 状态为 `Created/Ready/Running/Paused/Recovering` 时允许按策略写命令；`Sealed/Archived/Failed` 默认拒绝写入。
4. Replay 使用独立 ReplayRunId，禁止向原 Run 写入新的业务事实。
5. Command Accepted、Transfer Commit、Outbox 事实必须携带同一 RunId。

## 影响

- API、PLC Mapper、HIL Input、Recovery Worker 共用同一命令准入端口。
- Application 层负责 Run 状态前置校验，Domain/Runtime 仍必须二次校验，避免绕过入口。
- 测试必须覆盖跨来源相同 CommandId、Sealed Run、Replay 隔离和恢复重投。

## 反例

禁止控制器直接调用 `IDeviceController.ExecuteAsync`；禁止使用 HTTP RequestId 作为 CommandId；禁止从 Telemetry 重建命令事实。
