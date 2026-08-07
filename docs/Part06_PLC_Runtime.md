# IW.SIM Part06 PLC Runtime 架构设计

## 1. PLC Runtime 在 IW.SIM 中的定位

PLC Runtime 是 IW.SIM 工业机电控一体化仿真平台中用于复现工业控制器行为的核心运行模块。

它不是简单的数据模拟器，而是在仿真环境中复现真实 PLC 控制系统的运行语义，包括：

- 周期扫描机制
- 输入采集
- 用户程序执行
- 输出刷新
- 通讯处理
- 诊断报警
- 故障注入
- 与现场设备闭环控制

在实际立库项目中，WCS、PLC、设备之间形成如下控制链路：

```
WCS
 ↓
PLC Runtime
 ↓
Signal Runtime
 ↓
Device Runtime
 ↓
World Model
 ↓
Sensor Feedback
 ↓
PLC Runtime
```

IW.SIM 的目标不是模拟 PLC 界面，而是模拟 PLC 对自动化系统产生的控制行为。

---

# 2. PLC Runtime 总体架构

PLC Runtime 采用运行时内核 + 协议适配 + 控制逻辑插件架构。

```
                 PLC Runtime Host
                       |
        --------------------------------
        |              |               |
 Scan Scheduler   Memory Model   Logic Engine
        |              |               |
        |        Process Image     FB Runtime
        |        Tag Database      Ladder Runtime
        |                            Motion FB
        |
 Communication Runtime
        |
 S7 / OPC UA / TCP / Modbus
```

核心模块：

## 2.1 PLC Runtime Host

负责 PLC 实例生命周期管理：

- 创建 PLC
- 加载配置
- 启停运行
- 周期调度
- 状态管理

## 2.2 Scan Scheduler

负责模拟真实 PLC 扫描周期。

典型周期：

- 快速任务：1~10ms
- 普通任务：20~50ms
- 慢速任务：100ms+

仿真中所有周期基于 Simulation Kernel 提供的 SimTime，而不是操作系统时间。

---

# 3. PLC 扫描周期模型

真实 PLC 扫描过程：

```
输入刷新
   ↓
程序执行
   ↓
输出刷新
   ↓
通信服务
   ↓
下一扫描周期
```

IW.SIM 内部实现：

```
SimTime Tick
      |
PLC Scheduler
      |
Input Snapshot
      |
Logic Execute
      |
Output Commit
      |
Event Publish
```

这样保证：

- 仿真暂停后可以继续
- 快进运行可重复
- 自动测试结果一致

---

# 4. PLC 内存模型设计

PLC Runtime 需要模拟工业 PLC 地址空间。

支持：

- I 区（输入）
- Q 区（输出）
- M 区（内部变量）
- DB 数据块
- Symbol Tag

模型：

```
PlcMemory
 |
 +-- InputArea
 |
 +-- OutputArea
 |
 +-- MarkerArea
 |
 +-- DataBlockArea
```

示例：

```csharp
public class PlcTag
{
    public string Address {get;set;}
    public string Symbol {get;set;}
    public PlcDataType DataType {get;set;}
    public object Value {get;set;}
}
```

---

# 5. Process Image 过程映像设计

为了模拟 PLC 扫描一致性，引入过程映像。

```
现场设备
 ↓
Input Mapping
 ↓
Input Image
 ↓
PLC Logic
 ↓
Output Image
 ↓
Command Mapping
 ↓
设备
```

优势：

- 避免逻辑执行过程中数据变化
- 与真实 PLC 行为一致
- 支持扫描回放

---

# 6. PLC Logic Runtime

控制逻辑执行采用插件模型。

## 6.1 梯形图 Runtime

用于兼容传统自动化项目。

支持：

- 常开/常闭触点
- 线圈
- 定时器
- 计数器

## 6.2 Function Block Runtime

支持工业控制功能块：

- TON
- TOF
- CTU
- PID
- Motion Control FB

## 6.3 自定义逻辑 Runtime

允许使用 C# 编写特殊控制逻辑。

例如：

- 复杂输送逻辑
- 特殊设备控制
- 项目快速验证

---

# 7. PLC 与设备闭环模型

以输送机为例：

PLC 输出：

```
Motor_Start = TRUE
Speed_Set = 0.8m/s
```

Signal Runtime 转换：

```
Q100.0
 ↓
MotorCommand
```

Device Runtime 执行：

```
Conveyor.Move()
```

设备反馈：

```
Cargo Position
Sensor State
Motor Feedback
```

重新进入 PLC 输入区。

形成完整闭环。

---

# 8. PLC 通信仿真

支持真实工程联调：

## Siemens S7

支持：

- S7 TCP
- DB 数据访问
- Snap7 兼容
- TIA Portal 联调

## OPC UA

用于数字化系统集成。

## 自定义协议

支持项目现场私有 PLC 通讯协议。

---

# 9. 多 PLC 仿真架构

大型智能仓库通常存在：

- 输送 PLC
- 堆垛机 PLC
- RGV PLC
- 提升机 PLC

IW.SIM 支持多个独立 PLC Runtime：

```
Simulation Kernel
 |
 +-- PLC-01
 |      |
 |      Conveyor
 |
 +-- PLC-02
 |      |
 |      Stacker Crane
 |
 +-- PLC-03
        |
        RGV
```

每个 PLC：

- 独立变量空间
- 独立扫描周期
- 独立通信连接

---

# 10. 与 Simulation Kernel 集成

PLC Runtime 不拥有时间。

时间由 Simulation Kernel 管理：

```
Simulation Clock
       |
Event Scheduler
       |
PLC Scan Event
       |
PLC Runtime Execute
```

保证确定性。

---

# 11. .NET 10 工程实现建议

推荐结构：

```
src
 |
 +-- IW.Sim.Plc.Runtime
 |
 +-- IW.Sim.Plc.Memory
 |
 +-- IW.Sim.Plc.Logic
 |
 +-- IW.Sim.Plc.Protocol
 |
 +-- IW.Sim.Plc.Adapter
 |
 +-- IW.Sim.Plc.Test
```

核心接口：

```csharp
public interface IPlcRuntime
{
    void ExecuteCycle(SimTime time);
}
```

---

# 12. 与其他模块关系

|模块|职责|
|-|-|
|World Model|工业世界状态|
|Simulation Kernel|时间与事件|
|PLC Runtime|控制逻辑|
|Signal Runtime|信号转换|
|Device Runtime|设备行为|
|HIL Runtime|真实控制器接入|

---

# 13. 后续演进方向

- TIA Portal 工程解析
- 自动生成 PLC Tag
- IEC61131-3 Runtime
- Soft PLC
- AI PLC 调试助手
- 自动生成控制测试案例
