# PLC Runtime设计

## 1. 概述

PLC Runtime 是 IW.SIM 控制仿真的核心组件，用于模拟现场 PLC 扫描、逻辑执行、IO交互以及设备控制闭环。

## 2. 设计目标

- 支持真实 PLC 通信联调；
- 支持虚拟 PLC 运行环境；
- 保持现场控制逻辑一致性；
- 支持自动化测试和故障复现。

## 3. 核心组成

```text
PLC Runtime
 ├── Scan Cycle
 ├── Memory Area
 ├── IO Mapping
 ├── Logic Interface
 └── Communication Adapter
```

## 4. 扫描模型

模拟工业 PLC 的周期扫描：

1. 输入采集
2. 程序执行
3. 输出刷新
4. 通信同步

## 5. 与设备模型关系

设备 Runtime 提供状态和反馈，PLC Runtime 根据控制逻辑产生输出指令，形成完整闭环。