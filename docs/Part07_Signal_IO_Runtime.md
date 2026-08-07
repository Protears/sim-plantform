# IW.SIM Part07 Signal IO Runtime 架构设计

## 1. Signal IO Runtime 定位

Signal IO Runtime 是 IW.SIM 工业机电控一体化仿真平台中的现场信号抽象层，负责连接控制系统与设备模型。

它不是简单的 IO 点表管理，而是模拟工业现场从传感器、执行器、电气信号到 PLC 地址空间之间的完整数据链路。

核心职责：

- 数字量输入输出管理
- 模拟量采集与输出
- 信号状态转换
- PLC 地址映射
- 设备事件转换
- 信号诊断
- 故障注入

典型链路：

```
Device Runtime
      |
Sensor / Actuator
      |
Signal IO Runtime
      |
PLC Process Image
      |
PLC Logic
```

---

# 2. Signal IO 总体架构

```
Signal Runtime Host
 |
 +-- Signal Manager
 |
 +-- Mapping Engine
 |
 +-- Digital IO Runtime
 |
 +-- Analog IO Runtime
 |
 +-- Signal Filter
 |
 +-- Diagnostic Service
 |
 +-- Protocol Adapter
```

---

# 3. 信号模型设计

IW.SIM 中所有现场信号统一抽象为 Signal。

```csharp
public class Signal
{
    public string Id {get;set;}
    public SignalType Type {get;set;}
    public object Value {get;set;}
    public SignalState State {get;set;}
}
```

支持：

- Bool
- Int
- Real
- String
- Enum
- Structure

---

# 4. 数字量 IO 模型

典型工业信号：

输入：

- 光电传感器
- 接近开关
- 门禁信号
- 原点信号

输出：

- 电机启动
- 电磁阀
- 报警灯

模型：

```
Sensor
 |
Digital Input
 |
PLC I Address
```

---

# 5. 模拟量 IO 模型

支持：

- 速度给定
- 温度
- 压力
- 位移

提供：

- 原始值
- 工程值转换
- 量程校准
- 滤波

例如：

```
0-27648
    |
Scaling
    |
0-10m/s
```

---

# 6. IO Mapping 设计

针对立库项目，支持 Excel IO Mapping 导入。

映射关系：

```
设备信号
   |
Signal Mapping
   |
PLC Address
```

示例：

```
Conveyor01.Sensor01
        |
        I0.0
```

支持：

- 地址映射
- Symbol 映射
- 多 PLC 映射
- 版本管理

---

# 7. Signal 生命周期

```
Create
  |
Bind Device
  |
Mapping
  |
Runtime Update
  |
Publish Event
  |
Diagnostic
```

---

# 8. 与 Device Runtime 集成

设备产生状态：

```
Cargo Position
Motor State
Sensor Trigger
```

转换为：

```
Device Event
      |
Signal Runtime
      |
PLC Input
```

PLC 输出：

```
Q0.0 = TRUE
      |
Signal Runtime
      |
Motor Command
      |
Device Runtime
```

形成闭环控制。

---

# 9. 故障注入

支持工业测试场景：

- 传感器失效
- 信号抖动
- 延迟响应
- 常开故障
- 常闭故障
- 通讯断开

示例：

```
Sensor Fault
      |
Signal Override
      |
PLC Behavior Test
```

---

# 10. .NET 工程结构

```
IW.Sim.Signal.Runtime
IW.Sim.Signal.Model
IW.Sim.Signal.Mapping
IW.Sim.Signal.Protocol
IW.Sim.Signal.Test
```

核心接口：

```csharp
public interface ISignalRuntime
{
    void Update(SimTime time);
}
```

---

# 11. 与其他模块关系

|模块|职责|
|-|-|
|World Model|对象状态|
|Device Runtime|设备行为|
|Signal IO Runtime|现场信号|
|PLC Runtime|控制逻辑|
|Simulation Kernel|时间调度|

---

# 12. 后续演进

- TIA Portal IO 自动解析
- PLC 地址自动生成
- 电气回路仿真
- Safety IO 仿真
- AI 辅助 IO 调试
