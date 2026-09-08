# IW.SIM Part12 - HIL Simulation

## 1. 目标与边界

HIL（Hardware-in-the-Loop）用于把真实 PLC、运动控制器、现场 IO 或协议设备接入 IW.SIM，使控制程序面对与现场一致的信号、通信时序和故障边界，同时由仿真世界替代真实机械设备。

HIL 不是普通 TCP 转发，也不是让仿真 Runtime 直接依赖 PLC SDK。其边界为：

```text
Real Controller / PLC
        |
 Protocol Adapter
        |
 HIL Session + IO Contract
        |
 Signal / IO Runtime
        |
 Device Runtime
        |
 Simulation Kernel / World Model
```

核心原则：

- Simulation Kernel 不依赖具体 PLC 协议或硬件 SDK；
- 真实设备输入必须进入 Input Record，保证可追溯；
- 一个 HIL Session 只绑定一个明确的 Run；
- 外部墙上时钟不能直接决定 SimTick；
- IO 映射必须版本化并可验证；
- 断线、超时、协议异常必须显式进入 Session 状态机。

## 2. HIL 核心模型

```csharp
public sealed record HilSession(
    Guid SessionId,
    Guid RunId,
    string IntegrationProfileId,
    string IoContractVersion,
    HilSessionState State,
    long LastAppliedSimTick,
    DateTimeOffset ConnectedAt);

public enum HilSessionState
{
    Created,
    Connecting,
    Handshaking,
    Ready,
    Running,
    Degraded,
    Pausing,
    Paused,
    Recovering,
    Stopping,
    Stopped,
    Failed
}
```

Session 生命周期：

```text
Created -> Connecting -> Handshaking -> Ready -> Running
                                   |         |       |
                                   v         v       v
                                 Failed   Stopped  Degraded
                                                     |
                                                Recovering
                                                     |
                                                  Running
```

非法转换必须由 Command Handler 拒绝，不能由 UI 或协议适配器绕过。

## 3. 组件职责

| 组件 | 职责 |
|---|---|
| Protocol Adapter | 协议连接、编码、解码、重连 |
| HIL Session Manager | 生命周期、绑定 Run、状态转换 |
| IO Contract Validator | 映射版本、方向、数据类型、范围校验 |
| Signal Gateway | 输入采集和输出发布 |
| Time Synchronizer | 墙钟采样、延迟统计、Tick 对齐 |
| Safety Guard | 超时、心跳、危险输出、Fail-Safe |
| Input Recorder | 记录外部输入用于 Replay |
| Diagnostics | Trace、Metrics、Fault 关联 |

## 4. HIL 主流程

```text
CreateRun
  -> Resolve IntegrationProfile
  -> Validate IO Contract
  -> Create HilSession
  -> Connect Adapter
  -> Handshake
  -> Verify Capability / Mapping Hash
  -> Ready
  -> Start Simulation Run
  -> Running
       -> Read External Input
       -> Validate
       -> Timestamp / Sequence
       -> Input Record
       -> Schedule at SimTick
       -> Runtime executes
       -> Publish Output
       -> Ack / Timeout
```

输出不是“仿真状态变化后立即无限制发送”。Signal Gateway 必须根据 IO Contract 的刷新策略、边沿语义和 Session 状态决定是否发布。

## 5. 一致性与并发

同一 Run 同时只能有一个 Active HIL Session，使用唯一约束：

```text
UNIQUE(run_id) WHERE state IN (Connecting, Handshaking, Ready, Running, Degraded, Recovering)
```

同一 Session 的输入按 AdapterSequence 排序；重复输入由 `(SessionId, ExternalSequence)` 幂等键去重。协议没有序号时使用 `PayloadHash + TimeWindow` 仅作为降级策略，并必须记录去重策略。

## 6. 故障处理

- 连接失败：进入 Failed，Run 不启动；
- Handshake 失败：禁止进入 Ready；
- Mapping Hash 不一致：拒绝运行；
- 心跳超时：Running -> Degraded；
- 超过恢复窗口：Degraded -> Failed，并执行 Fail-Safe；
- 输出 ACK 超时：按信号 Criticality 决定重试、暂停或失败；
- Run 停止：先停止外部输出，再释放连接。

## 7. 可观测性

至少输出：

```text
hil_session_state
hil_input_latency_ms
hil_output_latency_ms
hil_connection_reconnect_total
hil_input_deduplicated_total
hil_contract_validation_failure_total
hil_last_applied_sim_tick
```

所有 HIL Trace 统一携带 `run.id`、`hil.session.id`、`integration.profile.id`、`io.contract.version`。

## 8. 验收

- Session 状态机覆盖所有连接/恢复/停止路径；
- 同一 Run 不出现两个 Active Session；
- Mapping 不一致不能启动；
- 重复输入不产生重复仿真刺激；
- 断线后行为符合 RunProfile；
- Replay 不重新连接真实硬件；
- Kernel 无 PLC SDK 依赖。
