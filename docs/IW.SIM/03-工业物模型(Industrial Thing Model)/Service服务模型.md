# IW.SIM 工业物模型 Service 服务模型

## 1. 概述

Service 是工业物模型中定义设备可执行能力的抽象，用于描述外部系统、控制器或人工操作对设备发起的动作请求。

Property 描述设备状态，Event 描述变化通知，而 Service 描述主动行为。

## 2. Service模型

```text
Service
 ├── Name
 ├── Description
 ├── Input Parameters
 ├── Output Result
 ├── Execution Policy
 └── Runtime Handler
```

## 3. 示例

输送机：

- Start
- Stop
- SetSpeed
- ResetFault

堆垛机：

- MoveHorizontal
- MoveVertical
- ForkAction

## 4. Runtime执行流程

```
Command
 ↓
Thing Service
 ↓
Capability Handler
 ↓
Device Runtime
 ↓
State/Event Update
```

## 5. 设计价值

Service模型实现设备能力统一抽象，使不同品牌设备可以通过统一接口参与仿真。