# IW.SIM Part09 - Digital Twin Integration

## 1. 文档定位

数字孪生集成层（Digital Twin Integration）是 IW.SIM 工业仿真平台连接“仿真世界模型”和“真实工业系统”的核心边界层。

本章节不定义三维可视化能力，也不替代仿真核心引擎，而是定义：

- 工业对象如何映射到数字孪生实体；
- 仿真状态如何同步到外部系统；
- 真实设备、PLC、WCS、MES 等系统如何接入仿真世界；
- 数字孪生数据如何形成闭环。

IW.SIM 的数字孪生理念：

> 一个可运行的工业世界模型，而不是静态设备模型展示系统。

---

# 2. 数字孪生总体架构

```
                 External Industrial Systems

 MES / WMS / WCS / PLC / SCADA / Robot Controller
                         |
                         |
              Digital Twin Integration Layer
                         |
 -------------------------------------------------
 | Twin Adapter | State Sync | Event Gateway |
 | Model Mapper | Data Bridge | Command Proxy |
 -------------------------------------------------
                         |
                 World Model Kernel
                         |
 -------------------------------------------------
 | Entity Model | Physics Model | Device Runtime |
 | Signal Model | Event Model  | Simulation Time |
 -------------------------------------------------
                         |
                Simulation Engine Runtime
```

数字孪生集成层负责协议、数据和模型之间的转换，但不包含设备运行逻辑。

---

# 3. 核心设计原则

## 3.1 Model First

所有数字孪生对象必须来自统一工业物模型。

例如输送机：

```text
Conveyor
 ├── Identity
 ├── Geometry
 ├── Capability
 ├── IO Mapping
 ├── Runtime State
 └── Events
```

外部系统看到的是工业对象，而不是仿真内部对象。

---

## 3.2 Runtime First

数字孪生实体必须绑定运行时状态。

例如：

```json
{
  "entityId": "CV001",
  "type": "Conveyor",
  "state": {
    "running": true,
    "speed": 0.8,
    "cargo": "LU10001"
  }
}
```

状态来源可以是：

- 仿真计算结果；
- PLC反馈数据；
- 真实设备采集数据。

---

## 4. Twin Entity 映射模型

IW.SIM采用 Entity Mapping 机制。

```
Simulation Entity
        |
        |
Twin Binding
        |
        |
External Entity
```

映射内容包括：

|类别|说明|
|-|-|
|Identity|唯一标识|
|Capability|能力模型|
|Telemetry|状态数据|
|Command|控制入口|
|Event|事件流|

---

# 5. 状态同步机制

## 5.1 双向同步

```
Simulation
    |
    | State Publish
    v
Digital Twin
    |
    | Command
    v
Simulation
```

支持模式：

### 仿真驱动模式

仿真作为唯一状态源。

用于：

- WCS算法验证；
- 自动化测试；
- 项目交付前联调。

### 真实设备镜像模式

现场设备作为状态源。

用于：

- 设备监控；
- 运维分析；
- 故障回放。

### Hybrid HIL模式

部分设备真实运行，部分设备仿真。

用于：

- PLC硬件在环测试；
- 控制系统验证。

---

# 6. Event Driven Integration

数字孪生不采用轮询同步作为主要机制。

核心采用事件驱动：

```
Device Runtime
      |
      v
Domain Event
      |
      v
Twin Event Bus
      |
      +---- WCS
      +---- Monitoring
      +---- Analytics
```

典型事件：

- CargoEnteredSensor
- DeviceStarted
- DeviceFaulted
- TaskCompleted
- PositionChanged

---

# 7. 数据架构

## Twin Data Categories

### Static Model

设备结构数据：

- Geometry
- Capability
- Configuration

### Runtime State

实时运行状态：

- Position
- Speed
- Occupancy
- Alarm

### Historical Data

历史分析：

- Event Log
- Operation Trace
- Simulation Replay

---

# 8. 与工业协议集成

Digital Twin Integration 支持协议适配：

```
Protocol Adapter

├── Siemens S7
├── OPC UA
├── MQTT
├── REST API
├── SignalR
└── Custom TCP
```

协议层只负责通信，不参与业务逻辑。

---

# 9. 与 IW.SIM Runtime 的边界

|模块|职责|
|-|-|
|World Model|工业世界描述|
|Simulation Engine|时间推进与事件执行|
|Device Runtime|设备行为模拟|
|Digital Twin Integration|外部系统连接|
|Visualization|呈现|

保持边界能够避免平台演变成单纯三维展示工具。

---

# 10. 典型应用场景

## WCS联调

WCS发送任务：

```
CreateTask
    |
    v
Twin Command Gateway
    |
    v
Simulation Runtime
    |
    v
Device Execution
```

验证：

- 调度策略；
- 异常处理；
- 设备协同。

---

## PLC Hardware In Loop

```
PLC
 |
 | S7 Protocol
 |
v
IW.SIM IO Runtime
 |
v
Digital Twin Model
```

实现真实控制程序驱动虚拟工厂。

---

# 11. 后续扩展方向

- Asset Administration Shell(AAS)兼容
- IEC 61499设备模型
- ROS2机器人系统集成
- MATLAB/Simulink联合仿真
- Unreal/Unity三维孪生渲染

---

# 12. 总结

IW.SIM Digital Twin Integration 的目标不是建立一个展示层，而是建立一个工业系统运行闭环：

> 从模型定义，到运行仿真，到外部控制，到状态反馈，形成可验证、可复现、可扩展的工业数字孪生基础设施。
