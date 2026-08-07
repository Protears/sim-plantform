# IW.SIM Part08 通信架构设计

## 1. 通信架构定位

在工业自动化仿真系统中，通信不是简单的数据传输，而是连接 WCS、PLC、设备控制器、现场协议和外部系统的工业网络运行环境。

IW.SIM 通信架构负责模拟真实项目中的通信行为，包括：

- PLC 与 WCS 通信
- PLC 与设备控制器通信
- 现场工业协议模拟
- 网络延迟与异常模拟
- 通信状态管理
- HIL 外部设备接入

整体链路：

```
WCS
 |
Communication Runtime
 |
PLC Runtime
 |
Signal Runtime
 |
Device Runtime
 |
World Model
```

---

# 2. 通信总体架构

采用 Adapter + Channel + Protocol 三层模型。

```
Communication Runtime
        |
+----------------+
| Channel Layer  |
+----------------+
        |
+----------------+
| Protocol Layer |
+----------------+
        |
+----------------+
| Adapter Layer  |
+----------------+
```

核心职责：

- 建立连接
- 数据编码解码
- 消息路由
- 状态监控
- 异常处理

---

# 3. Channel 通道模型

通信通道抽象真实工业连接。

支持：

- TCP Channel
- UDP Channel
- Serial Channel
- Shared Memory Channel
- Virtual Channel

模型：

```csharp
public interface ICommunicationChannel
{
    Task SendAsync(byte[] data);
    Task<byte[]> ReceiveAsync();
    ConnectionState State {get;}
}
```

---

# 4. 协议适配模型

IW.SIM 不绑定单一协议，通过协议插件扩展。

支持：

## Siemens S7

用于 PLC 联调：

- S7 TCP
- DB 数据访问
- Snap7 兼容

## OPC UA

用于数字化系统集成。

## Modbus TCP

用于标准设备通信。

## 自定义 TCP 协议

支持现场设备私有协议。

---

# 5. S7 通信仿真设计

针对立库项目，S7 是核心协议。

结构：

```
External PLC
      |
 S7 Client
      |
 IW.SIM S7 Server
      |
 PLC Memory Model
```

S7 Server 映射：

```
DB Area
 |
PlcTag
 |
Signal Mapping
 |
Device State
```

支持：

- DB 读写
- I/Q 区映射
- 地址转换
- 数据一致性

---

# 6. WCS 通信模型

WCS 联调是 IW.SIM 核心场景。

支持：

```
WCS
 |
Task Command
 |
Communication Runtime
 |
PLC Runtime
 |
Equipment
```

包括：

- 任务下发
- 状态反馈
- 心跳
- 报警
- 完成通知

---

# 7. 通信异常仿真

为了接近现场环境，支持：

- 网络延迟
- 丢包
- 重连
- 超时
- 数据错误
- 服务不可用

模型：

```
Message
 |
Fault Simulator
 |
Network Condition
 |
Receiver
```

---

# 8. 与 Simulation Kernel 集成

通信事件由仿真时间驱动。

```
Simulation Clock
       |
Communication Event
       |
Message Queue
       |
Protocol Handler
```

保证：

- 可暂停
- 可快进
- 可回放
- 可测试

---

# 9. .NET 工程结构

推荐：

```
IW.Sim.Communication.Runtime
IW.Sim.Communication.Channel
IW.Sim.Communication.Protocol
IW.Sim.Communication.S7
IW.Sim.Communication.OpcUa
IW.Sim.Communication.Test
```

核心接口：

```csharp
public interface IProtocolAdapter
{
    Task HandleAsync(MessageContext context);
}
```

---

# 10. 与其他模块关系

|模块|职责|
|-|-|
|Simulation Kernel|时间调度|
|Communication Runtime|通信行为|
|PLC Runtime|控制逻辑|
|Signal Runtime|信号转换|
|Device Runtime|设备执行|
|WCS|业务调度|

---

# 11. 后续演进

- TIA Portal 在线联调
- OPC UA Server
- MQTT 工业物联网接入
- 网络拓扑仿真
- 通信性能分析
- AI 自动诊断通信异常
