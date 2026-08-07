# IW.SIM Part06 - PLC Runtime Architecture Design

## 1. PLC Runtime Overview

PLC Runtime 是 IW.SIM 工业仿真平台中负责控制器行为仿真的核心运行时，用于在无真实 PLC 或与真实 PLC 并行运行的情况下，复现工业控制系统的扫描周期、变量刷新、逻辑执行、通信交互和故障行为。

PLC Runtime 不等同于简单 PLC 数据模拟器，而是一个面向工业控制语义的可确定性执行环境。

核心职责：

- PLC 扫描周期调度
- 输入镜像区刷新
- 用户逻辑执行
- 输出镜像区更新
- 通信变量同步
- 状态诊断与故障注入
- 与 Device Runtime 联动

## 2. Runtime Architecture

PLC Runtime 分为以下层次：

```
PLC Runtime Host
 ├── Scan Scheduler
 ├── Process Image Manager
 │    ├── Input Image
 │    └── Output Image
 ├── Logic Execution Engine
 │    ├── Ladder Runtime
 │    ├── Function Block Runtime
 │    └── Custom Logic Runtime
 ├── Communication Adapter
 ├── Diagnostic Manager
 └── Simulation Event Adapter
```

## 3. PLC Scan Cycle Model

工业 PLC 的核心模型是周期扫描：

```
Input Refresh
      ↓
Program Execute
      ↓
Output Update
      ↓
Communication Service
      ↓
Next Cycle
```

IW.SIM 中通过 Simulation Kernel 提供统一 SimTime，PLC Runtime 不直接依赖系统时间。

```csharp
public interface IPlcRuntime
{
    void Scan(SimTime time);
    PlcState State { get; }
}
```

## 4. Process Image Design

PLC Runtime 使用过程映像区模型隔离设备实时状态与控制逻辑。

```
Physical Device
       |
Signal Mapping
       |
Input Image
       |
PLC Logic
       |
Output Image
       |
Device Command
```

支持：

- Bool
- Byte
- Word
- DWord
- Int
- Real
- String
- Structure

## 5. PLC Variable Model

PLC Tag 是控制系统中的核心数据对象。

```csharp
public class PlcVariable
{
    public string Address {get;set;}
    public PlcDataType Type {get;set;}
    public object Value {get;set;}
    public DateTime SimTimestamp {get;set;}
}
```

支持地址：

- I/O 地址
- DB 地址
- Marker 地址
- Memory 地址
- Symbol 地址

## 6. Logic Execution Engine

逻辑执行引擎采用插件化设计。

支持：

### Ladder Runtime

用于复现传统 PLC 梯形图逻辑。

### Function Block Runtime

支持：

- TON
- TOF
- CTU
- PID
- Motion FB

### Custom Runtime

允许使用 C# 编写仿真控制逻辑。

## 7. PLC Communication Layer

PLC Runtime 提供工业协议适配：

```
Communication Runtime
        |
 ┌───────────────┐
 │ S7 Adapter    │
 │ OPC UA        │
 │ Modbus        │
 │ Custom TCP    │
 └───────────────┘
```

用于支持：

- WCS 联调
- TIA Portal 联调
- 真实 PLC HIL

## 8. Device Runtime Integration

PLC Runtime 与 Device Runtime 通过 Signal / Command Event 交互。

控制链路：

```
PLC Output
    ↓
Signal Runtime
    ↓
Device Command
    ↓
Device Runtime
    ↓
World State Update
```

状态反馈：

```
Device State
    ↓
Sensor Signal
    ↓
PLC Input Image
    ↓
PLC Logic
```

## 9. Deterministic Execution

为了保证仿真可重复：

- 禁止直接使用 DateTime.Now
- 所有时间来自 SimClock
- 固定 Scan Cycle
- Event Queue 顺序确定

支持：

- 回放测试
- 自动化测试
- 故障复现

## 10. Multi PLC Architecture

大型立库通常包含多个 PLC。

IW.SIM 支持：

```
Simulation Kernel
        |
 -----------------
 |       |        |
PLC-01 PLC-02 PLC-03
 |       |        |
Device Device Device
```

每个 PLC 拥有：

- 独立扫描周期
- 独立变量空间
- 独立通信通道

## 11. Engineering Implementation (.NET)

推荐工程结构：

```
src
 ├── IW.Sim.Plc.Runtime
 ├── IW.Sim.Plc.Protocol
 ├── IW.Sim.Plc.Logic
 ├── IW.Sim.Plc.Model
 └── IW.Sim.Plc.Test
```

核心接口：

```csharp
public interface IPlcScanEngine
{
    Task ExecuteCycleAsync(SimTime time);
}
```

## 12. Relationship With Other Modules

| Module | Responsibility |
|-|-|
| World Model | 世界状态 |
| Simulation Kernel | 时间与事件调度 |
| PLC Runtime | 控制逻辑 |
| Signal Runtime | 信号交换 |
| Device Runtime | 设备行为 |
| HIL Runtime | 外部控制器接入 |

## 13. Future Extension

后续支持：

- TIA Portal 工程解析
- S7 PLC Binary Runtime
- IEC61131-3 Runtime
- Soft PLC
- PLC AI Debug Assistant
- 自动生成测试场景
