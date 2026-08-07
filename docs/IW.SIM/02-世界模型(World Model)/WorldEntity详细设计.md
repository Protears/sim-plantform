# IW.SIM WorldEntity详细设计

## 1. 概述

WorldEntity 是 IW.SIM 世界模型中的基础运行实体，是所有工业对象在仿真环境中的统一抽象。

设备、货载、空间节点、传感器等对象均通过 WorldEntity 纳入统一管理。

## 2. 核心属性

WorldEntity包含：

- Identity：唯一标识
- Type：实体类型
- Transform：空间位置与姿态
- State：运行状态
- Properties：扩展属性
- Relations：对象关系
- Behaviors：行为集合

## 3. 实体关系

```
World
 └── Entity
      ├── Device
      ├── LoadUnit
      ├── Sensor
      ├── Place
      └── Controller
```

## 4. 生命周期

实体生命周期包括：

1. 创建
2. 初始化
3. 注册Runtime
4. 状态演化
5. 销毁

## 5. 设计原则

WorldEntity不绑定具体设备协议，通过领域模型承载工业对象语义，由Runtime驱动行为变化。
