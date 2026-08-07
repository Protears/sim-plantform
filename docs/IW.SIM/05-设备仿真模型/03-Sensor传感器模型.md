# Sensor传感器模型设计

## 1. 概述

传感器模型用于模拟工业现场检测元件，是设备模型与PLC控制之间的重要连接。

## 2. 传感器类型

支持：

- 光电传感器
- 接近传感器
- RFID检测
- 位置检测

## 3. 模型属性

```text
Sensor
 ├── Identity
 ├── Position
 ├── Direction
 ├── DetectionRange
 ├── State
 └── IO Address
```

## 4. 触发机制

当Cargo进入检测区域：

```
Cargo Position Change
        ↓
Sensor Collision Check
        ↓
Sensor State Update
        ↓
PLC Signal Output
```

## 5. 工业场景支持

支持多传感器重叠场景，例如：

- 前后检测点同时存在；
- 货载遮挡多个传感器；
- 延迟释放。