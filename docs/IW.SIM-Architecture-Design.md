# IW.SIM 整体架构设计文档

## 1. 概述

IW.SIM（Industrial Warehouse Simulation Platform）是一套面向物流自动化装备的机电控一体化仿真与联调平台，用于构建立库、输送系统、RGV、堆垛机、PLC 控制系统及 WCS 调度系统的数字化仿真环境。

平台目标：

- 支持 WCS 软件调度算法验证
- 支持 PLC 程序离线调试
- 支持设备协议一致性验证
- 支持现场问题复现与自动化测试
- 支持数字孪生和 HIL（Hardware In Loop）扩展

## 2. 总体架构

IW.SIM 采用分层模块化架构：

```
+------------------------------------------------+
| Application Layer                              |
| Web UI / Scene Editor / Test Platform / API    |
+------------------------------------------------+
| Control Integration Layer                      |
| PLC Runtime / WCS Adapter / Protocol Adapter  |
+------------------------------------------------+
| Simulation Runtime                             |
| World Kernel / Event Engine / Device Runtime  |
+------------------------------------------------+
| Modeling Layer                                 |
| Thing Model / Space Model / Device Model      |
+------------------------------------------------+
| Infrastructure Layer                           |
| Database / Message / Logging / Monitoring     |
+------------------------------------------------+
```

## 3. 技术架构

### 后端

- .NET 10
- C# 13
- ASP.NET Core
- EF Core
- PostgreSQL
- SignalR
- OpenTelemetry

### 前端

- Vue 3
- TypeScript
- Vite
- PixiJS

## 4. 核心模块设计

## 4.1 World Kernel

世界模型核心，用于维护仿真世界状态。

职责：

- 实体生命周期管理
- 空间关系维护
- 状态快照
- 时间推进
- 查询服务

核心对象：

- World
- Entity
- Thing
- Component
- Relation

## 4.2 Simulation Runtime

基于离散事件驱动仿真模型。

核心能力：

- SimTime 仿真时间
- Event Scheduler 事件调度
- Tick Loop
- Deterministic Simulation

典型周期：30ms。

## 4.3 Device Runtime

负责设备行为模拟。

支持：

- Conveyor Runtime
- RGV Runtime
- Stacker Crane Runtime
- Robot Runtime
- Sensor Runtime
- IO Runtime

设备模型采用能力驱动：

```
Device
 ├── Capability
 ├── State
 ├── Command
 ├── Sensor
 └── Event
```

## 4.4 PLC Runtime

提供 PLC 控制环境模拟。

支持：

- Siemens S7 协议
- IO Mapping
- PLC Scan Cycle
- DB 数据区模拟
- 信号同步

## 4.5 WCS Integration

提供 WCS 联调接口。

支持：

- REST API
- WebSocket/SignalR
- TCP 协议适配
- 任务下发
- 状态反馈

## 5. 工业物模型设计

采用 Thing Model 思想。

```
Thing
 ├── Properties
 ├── Services
 ├── Events
 └── Capabilities
```

示例：输送机

Properties:

- speed
- position
- loadState

Services:

- Start
- Stop
- Reset

Events:

- CargoArrived
- Fault

## 6. 空间模型

支持物流自动化场景建模：

```
Site
 └── Warehouse
      ├── Area
      ├── Aisle
      ├── Rack
      ├── Track
      └── Device
```

核心实体：

- LoadUnit
- Cargo
- Place
- RackGrid
- Track

## 7. 输送系统仿真

支持：

- 多段输送机
- 货载追踪
- 传感器触发
- 加减速运动模型
- PLC 点位映射

运动模型：

- Uniform Motion
- Uniform Accelerated Motion

单位：

- 速度：m/s
- 加速度：m/s²
- 距离：m

## 8. 数据架构

数据分类：

### 配置数据

- 场景配置
- 设备模型
- PLC 映射

### 运行数据

- 实时状态
- 事件流
- 仿真日志

### 历史数据

- Event Log
- Trace
- Replay Data

## 9. 可观测性

采用 OpenTelemetry：

支持：

- Trace
- Metric
- Log

集成：

- Prometheus
- Grafana
- Loki

## 10. 测试体系

支持自动化测试：

- 场景测试
- PLC 联调测试
- WCS 调度测试
- 回归测试

## 11. 扩展方向

未来支持：

- 数字孪生实时同步
- ROS2 集成
- MATLAB/Simulink 联合仿真
- Unity/Unreal 三维可视化
- AI Agent 自动测试

## 12. 设计原则

- Domain First
- Model Driven
- Event Driven
- Deterministic Simulation
- Modular Architecture
- Industrial Protocol Compatibility

---

IW.SIM 是面向智能制造和物流自动化领域的新一代机电控一体化仿真基础平台。
