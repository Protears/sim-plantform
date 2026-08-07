# Part 01 - IW.SIM Architecture Overview

## 1. 平台定位

IW.SIM 是面向物流自动化、智能仓储及工业控制系统的机电控一体化仿真平台。

平台目标是构建从设备模型、控制系统、工业协议到业务调度系统的统一仿真环境，实现：

- 设备级仿真
- PLC 控制联调
- WCS/WMS 系统验证
- 数字孪生数据支撑
- 硬件在环（HIL）测试

## 2. 总体架构原则

采用领域驱动设计（DDD）、模块化架构和事件驱动架构。

核心设计原则：

1. World Model First
2. Simulation Kernel 驱动时间演进
3. Thing Model 统一工业对象描述
4. Runtime 模拟真实工业执行环境
5. Protocol Adapter 保持现场一致性

## 3. 分层架构

### 3.1 World Model Layer

负责描述仿真世界中的实体、空间、关系和状态。

主要对象：

- World
- Entity
- Thing
- Device
- Resource
- LoadUnit
- Space

### 3.2 Simulation Layer

负责离散事件仿真、时间管理和状态推进。

核心组件：

- Simulation Clock
- Event Scheduler
- Simulation Kernel
- Deterministic Runtime

### 3.3 Runtime Layer

模拟工业现场运行时环境：

- PLC Runtime
- Device Runtime
- Drive Runtime
- Signal IO Runtime

### 3.4 Integration Layer

提供外部系统连接能力：

- WCS
- PLC
- OPC UA
- S7 Protocol
- MQTT
- REST API

## 4. 技术方向

目标技术栈：

- .NET 10 / C# 13
- ASP.NET Core
- EF Core
- PostgreSQL
- Vue 3 + TypeScript
- PixiJS
- OpenTelemetry

## 5. 后续章节

后续将继续提交 World Model、Simulation Kernel、Thing Model 等详细设计章节。
