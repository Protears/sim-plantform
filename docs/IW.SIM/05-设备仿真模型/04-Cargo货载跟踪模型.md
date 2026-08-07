# Cargo货载跟踪模型设计

## 1. 概述

Cargo模型描述物流系统中的载货单元，是IW.SIM世界模型的重要实体。

## 2. 核心属性

```text
Cargo
 ├── Id
 ├── LoadUnit
 ├── Size
 ├── Position
 ├── Velocity
 ├── CurrentDevice
 └── State
```

## 3. 跟踪机制

货载位置由设备运动驱动：

```
Device Runtime
      ↓
Motion Update
      ↓
Cargo Position Update
      ↓
World Model Update
```

## 4. 业务状态

支持：

- 创建
- 输送
- 暂存
- 入库
- 出库
- 异常

## 5. 应用

用于：

- WCS调度验证
- 货流分析
- 设备联动测试
- 数字孪生展示
