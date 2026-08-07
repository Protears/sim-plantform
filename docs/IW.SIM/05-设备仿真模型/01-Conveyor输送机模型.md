# Conveyor输送机模型设计

## 1. 概述

输送机是物流自动化系统中最基础的输送设备，也是 IW.SIM 设备仿真体系的重要组成部分。

模型不仅描述几何结构，还需要描述运动、控制、传感器、货载交互以及 PLC 信号行为。

## 2. 核心组成

```text
Conveyor
 ├── Segment
 ├── Motor
 ├── Drive
 ├── Sensor
 ├── Cargo
 ├── IO Mapping
 └── State Machine
```

## 3. Segment模型

输送线采用多段模型，每个 Segment 具有独立运行能力。

属性包括：

- 长度
- 运行方向
- 最大速度
- 加速度
- 当前货载
- 控制状态

## 4. 行为模型

支持：

- 启动
- 停止
- 加速
- 减速
- 急停
- 故障

## 5. 控制接口

设备通过抽象IO与PLC交互：

- Run Command
- Stop Command
- Sensor Signal
- Fault Feedback
- Ready Status

## 6. 设计目标

保证仿真设备行为与现场设备控制逻辑一致，为WCS联调和PLC虚拟调试提供基础。