# RGV设备仿真模型设计

## 1. 概述

RGV（Rail Guided Vehicle）是物流自动化系统中典型的轨道运输设备。IW.SIM中RGV不是简单的位置对象，而是具备运动能力、任务能力、控制接口和状态演化能力的工业实体。

## 2. 模型组成

RGV由以下核心部分组成：

- 车体模型（Vehicle Body）
- 轨道模型（Rail Track）
- 驱动模型（Drive）
- 定位模型（Positioning）
- 任务执行模型（Task Execution）
- PLC接口模型（PLC Interface）

## 3. 运动模型

支持：

- 匀速运行
- 加减速运行
- 目标点定位
- 到站检测
- 故障停止

运动过程由Simulation Runtime驱动。

## 4. 控制闭环

```
WCS任务
 ↓
RGV Runtime
 ↓
PLC Command
 ↓
IO反馈
 ↓
状态更新
```

## 5. 状态模型

包含：

- Idle
- Moving
- Loading
- Unloading
- Fault
- EmergencyStop
