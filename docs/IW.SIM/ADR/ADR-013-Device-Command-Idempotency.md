# ADR-013：设备命令幂等与重试语义

- 状态：Accepted
- 适用：Part05 Device Runtime、Part10 Data、Part11 Application

## 决策

1. `CommandId` 是业务幂等键；网络重试、进程重启和 Outbox 重投均复用同一 CommandId。
2. 设备执行结果写入 Inbox/CommandResult 索引，重复请求直接返回原结果。
3. 只有未到 Executing 的命令允许取消；执行中取消必须产生显式 CancelRequested 事件。
4. 失败重试由策略决定，但不能改变原始命令的因果链。
5. Sealed Run 禁止重新执行命令，只允许查询和 Replay。

## 数据约束

```text
UNIQUE(run_id, command_id)
UNIQUE(run_id, command_id, result_version)
INDEX(run_id, device_id, status, target_sim_tick)
```

## 影响

Application 可安全重试提交；Device Runtime 不需要猜测调用方是否重复；Part10 可完整记录 causation/correlation；测试必须覆盖超时后重投和恢复后重投。
