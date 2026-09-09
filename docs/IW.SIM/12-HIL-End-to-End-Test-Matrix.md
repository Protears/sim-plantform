# Part 12：HIL 端到端验收测试矩阵

| 编号 | 前置条件 | 操作 | 验证点 | 结果 |
|---|---|---|---|---|
| HIL-E2E-001 | 合同 Hash 一致 | 建立连接并握手 | Ready，Epoch=1 | 必须通过 |
| HIL-E2E-002 | 合同 Hash 不一致 | 握手 | 返回 HIL-COM-006，不进入 Ready | 必须通过 |
| HIL-E2E-003 | 已连接 | 连续发送同一 ExternalSequence | 仅一条 InputRecord | 必须通过 |
| HIL-E2E-004 | 已运行 | 发送迟到输入 | 按策略 Reject/Clamp/NextTick 且可审计 | 必须通过 |
| HIL-E2E-005 | 已运行 | 断开 2 个心跳周期 | Degraded，关键输出进入安全策略 | 必须通过 |
| HIL-E2E-006 | Degraded | 恢复连接 | Epoch 递增，执行状态对账 | 必须通过 |
| HIL-E2E-007 | 已运行 | 输出无 Ack | Critical 输出在安全窗口内 FailSafe | 必须通过 |
| HIL-E2E-008 | 已运行 | 重放输入 | 与原运行 Event/State/Result Hash 一致 | 必须通过 |
| HIL-E2E-009 | 高负载 | 队列达到 100% | 非关键输入按策略丢弃并计数，关键输入触发故障 | 必须通过 |
| HIL-E2E-010 | 半包/粘包 | 发送拆分帧 | 正确重组且不产生重复信号 | 必须通过 |

## 通过门槛

- 10 项全部通过；任一关键安全场景失败则整体不通过。
- 运行 72 小时无未分类异常、无事件序列倒退、无内存持续增长。
- 任一测试失败必须保留 `SessionId/RunId/CorrelationId/ExternalSequence/ClockEpoch`。
