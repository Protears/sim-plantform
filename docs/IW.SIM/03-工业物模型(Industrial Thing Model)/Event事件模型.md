# IW.SIM 工业物模型 Event事件模型

## 1. 概述

Event用于描述工业对象运行过程中的状态变化，是连接设备Runtime、PLC、WCS和上层应用的重要机制。

## 2. 事件分类

### 生命周期事件

- Created
- Started
- Stopped
- Destroyed

### 状态事件

- PositionChanged
- StateChanged
- FaultOccurred

### 业务事件

- CargoArrived
- TaskCompleted
- StorageFinished

## 3. 事件结构

```text
Event
 ├── EventId
 ├── Timestamp
 ├── Source
 ├── Type
 ├── Payload
 └── CorrelationId
```

## 4. 应用

事件驱动机制支持：

- 仿真状态同步
- PLC反馈
- WCS任务流转
- 历史追踪
