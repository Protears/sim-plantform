# IW.SIM 状态演化 Pipeline 设计

## 1. 概述

状态演化 Pipeline 是 IW.SIM 中连接事件、行为和世界状态的核心流程。

## 2. 状态演化流程

```
Input
 ↓
Command
 ↓
Event
 ↓
Behavior
 ↓
State Change
 ↓
World Model Update
 ↓
Observation
```

## 3. 设计原则

### 单向演化

状态变化只能通过合法行为产生。

### 可追踪

每次变化产生对应事件记录。

### 可回放

通过事件序列恢复历史状态。

## 4. 典型案例

输送机运行：

PLC启动信号 → 电机状态变化 → Cargo位置更新 → Sensor触发 → IO反馈。

该模型保证设备仿真与真实控制系统行为一致。
