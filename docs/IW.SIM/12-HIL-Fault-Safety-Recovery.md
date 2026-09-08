# Part12 HIL - 故障、安全与恢复设计

## 故障模型

```text
ProtocolFault
ConnectionFault
ContractFault
TimingFault
InputFault
OutputFault
SafetyFault
PersistenceFault
```

故障事件统一包含：FaultId、RunId、SessionId、Category、Severity、OccurredAt、SimTick、CorrelationId、RecoveryPolicy。

## 恢复状态机

```text
Healthy
  -> Suspected
  -> Degraded
  -> Recovering
  -> Healthy
        \-> Failed
```

Critical Fault 不允许自动恢复后继续执行未验证的输出。恢复流程：

```text
Detect
 -> Block Unsafe Output
 -> Record Fault
 -> Snapshot/Checkpoint Reference
 -> Reconnect
 -> Revalidate Contract
 -> Resynchronize Clock
 -> Reconcile Signal State
 -> Explicit Resume
```

其中 `Reconcile Signal State` 必须比较控制器当前输入、Runtime State 和最后确认输出；发现不可判定差异时进入 Paused 等待人工策略，而不是猜测覆盖。

## Fail-Safe

每个 Critical Output 必须定义：

```text
SafeValue
MaxStaleDuration
RequiredAck
RecoveryApproval
```

Session 进入 Failed/Stopping 时按确定顺序执行 Fail-Safe，执行结果记录为独立 Safety Event。

## 与数据架构一致性

HIL 外部输入写入 Input Record；HIL 故障写入 Simulation Event；Telemetry 仅作为观测，不作为恢复事实来源。恢复点必须引用 Part10 Snapshot 的 `BaseEventSequence`，不得通过日志推断状态。

## 测试矩阵

1. 网线拔出；
2. 半连接/心跳丢失；
3. PLC 重启导致 Session 重置；
4. Contract Hash 不一致；
5. 输入乱序和重复；
6. 输出 ACK 超时；
7. 时钟跳变；
8. Snapshot 存储失败；
9. PersistenceDegraded；
10. Recovering 期间收到新输入。

验收：所有场景必须有确定状态、错误码、事件记录和恢复/终止路径；不存在“异常被吞掉后继续运行”。
