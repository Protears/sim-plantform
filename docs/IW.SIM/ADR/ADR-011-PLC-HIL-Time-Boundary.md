# ADR-011：PLC Runtime 与 HIL 的时间边界

- 状态：Accepted
- 日期：2026-09-10
- 适用范围：Part03 Simulation Kernel、Part06 PLC Runtime、Part07 Signal/IO、Part08 Communication、Part12 HIL

## 背景

PLC 扫描、现场协议帧和仿真事件具有不同时间来源。若使用 WallClock 直接驱动扫描或以接收线程完成顺序提交输入，会破坏确定性、回放一致性和故障恢复。

## 决策

1. Simulation Kernel 是唯一 `SimTick` 权威。
2. PLC 扫描只由 `SimTick + PeriodTicks + PhaseTicks` 触发。
3. HIL 输入必须携带或推导 `ExternalSequence、ClockEpoch、TargetSimTick`。
4. 输入在 `InputCommit` 前统一进入 Part10 InputRecord，再由 Kernel 在 TickBoundary 注入。
5. PLC 输出在 `OutputCommit` 后才能发送到 HIL；协议 Ack 不得阻塞 Kernel。
6. 重连必须递增 `ClockEpoch`，旧 Epoch 数据不具备业务效力。
7. Replay 不读取原始 WallClock，只消费已记录的输入和调度顺序。

## 影响

- 优点：确定性、可回放、可测试、可诊断。
- 代价：需要维护 InputRecord、Epoch、Process Image Hash 和 Ack 关联。
- 约束：协议适配器、Transport、PLC Runtime 不得自行推进仿真时间。

## 违反决策的判定

以下任一情况即视为架构违规：

- PLC Runtime 调用 `DateTime.UtcNow` 决定扫描结果。
- HIL 接收线程直接修改 World State。
- Output Ack 在 Kernel 主循环同步等待。
- 断线重连继续使用旧 ClockEpoch。

## 验收标准

相同 RunProfile、初始 Snapshot 和 InputRecord 重放 100 次，PLC Output Hash、World State Hash、Result Hash 必须一致；网络延迟变化不得改变逻辑结果，只能改变诊断指标。
