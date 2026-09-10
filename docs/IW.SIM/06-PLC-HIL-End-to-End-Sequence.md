# PLC-HIL-设备端到端时序与工程任务拆解

## 1. 端到端主链路

```text
HIL Transport
 -> Protocol Adapter
 -> HilSession/Clock Normalizer
 -> InputRecord
 -> Simulation Kernel TickBoundary
 -> Signal IO InputCommit
 -> PLC Process Image
 -> PLC Logic Execute
 -> OutputCommit
 -> Device Runtime
 -> World Model/Cargo/Sensor Event
 -> Signal IO Feedback
 -> InputRecord(next cycle)
```

## 2. 正常时序

|序号|组件|动作|必须产生的事实|
|-:|---|---|---|
|1|Transport|接收原始帧|TransportSequence|
|2|Adapter|解码并校验|DecodeResult、ProtocolVersion|
|3|HIL Session|补充 Epoch/Tick|HilInputStamp|
|4|Data|写入输入记录|ExternalSequence 唯一|
|5|Kernel|在 TickBoundary 取数|InputBatchHash|
|6|Signal IO|提交输入映像|InputImageVersion|
|7|PLC|执行扫描|CycleSequence、OutputImageHash|
|8|Signal IO|提交输出映像|OutputImageVersion|
|9|Device|执行命令|DeviceCommandApplied|
|10|World Model|更新占用/位置|WorldStateVersion|
|11|Sensor|计算反馈|SensorChanged|
|12|HIL|发布输出/反馈|Ack 或 Fault|

## 3. 关键异常时序

### 3.1 输入迟到

`ReceiveTime > TargetSimTick` 时，根据 RunProfile 选择：拒收、下一个 Tick、或暂停等待。无论策略如何，必须记录 `LateByTicks`，不得静默修正。

### 3.2 PLC 扫描超时

LogicExecute 超过 `MaxExecutionTicks`：丢弃本周期 StagingOutput，保留上一有效 OutputImage，发布 `PlcScanOverrun`，并按关键输出策略进入安全态。

### 3.3 HIL 断线

Transport Lost → Adapter Degraded → HIL Session Reconnecting → ClockEpoch++ → Handshake → Signal Reconciliation。对账完成前禁止恢复普通输出，只允许安全输出和诊断帧。

## 4. 可直接拆分的工程任务

1. 实现 `IPlcScanScheduler` 与固定排序器。
2. 实现 `IProcessImage` 双缓冲及 Hash Commit。
3. 实现 `IProtocolAdapter` 半包/粘包重组适配测试。
4. 实现 `InputRecord` 幂等键 `(RunId, SessionId, ClockEpoch, ExternalSequence)`。
5. 实现 `ClockEpoch` 递增和旧帧拒收中间件。
6. 实现 Critical Output Ack 超时安全策略。
7. 实现 `PlcScanOverrun`、`InputRejected`、`OutputSafeStateApplied` 事件。
8. 实现跨模块 CorrelationId 传播和结构化日志。

## 5. 验收门槛

- 100 个 PLC、每个 2 个周期任务运行 72 小时无未处理异常。
- 30ms 主 Tick 下，Kernel 漂移不超过 60ms。
- 同一输入记录重复投递不产生第二次设备命令。
- 断线重连后的首个普通输出必须晚于状态对账完成事件。
- 回放模式中网络延迟、线程调度和 WallClock 改变不影响结果 Hash。
