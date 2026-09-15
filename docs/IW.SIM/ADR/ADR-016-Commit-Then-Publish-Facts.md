# ADR-016：事实先提交、事件后发布

## 状态

Accepted

## 背景

设备命令、Occupancy Transfer、Outbox 和 SignalR 需要在数据库、Kernel 和外部消费者之间保持一致。若先发布后提交，消费者可能观察到最终不存在的事实；若由消息确认决定业务成功，又会把网络延迟引入确定性仿真。

## 决策

采用“Commit Then Publish”：

1. 在同一事务内完成命令/Transfer 状态、事实表和 Outbox 写入。
2. 事务提交后由 Dispatcher 发布 Outbox。
3. 消费者使用 Inbox/EventId 幂等。
4. Kernel 不等待发布确认，不因外部发布失败回滚已提交仿真事实。

## 影响

- 正确性以数据库提交为准。
- 事件至少一次投递，消费端必须幂等。
- 需要 Pending、Leased、Published、DeadLetter 状态和租约字段。
- Replay 使用已提交事实和事件，不使用消息中间件时间顺序。

## 约束

- Outbox 不得包含未提交业务状态。
- Dispatcher 不得直接执行业务动作。
- SignalR 仅为投影通道，不能作为恢复真相。
- 任何绕过 Outbox 的事实事件发布都视为架构违规。

## 验收

- 数据库回滚时消费者收不到 Accepted/Committed。
- Dispatcher 重试不重复执行设备动作。
- EventId 重投不会重复修改投影。
- 断线补发后 Sequence 连续且顺序可验证。
