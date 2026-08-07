# IW.SIM Thing模型设计

## 1. Thing定义

Thing表示一个独立存在于工业世界中的对象。

它可以是真实设备，也可以是虚拟对象。

## 2. 基础结构

```
Thing
 ├── Identity
 ├── Metadata
 ├── Properties
 ├── Capabilities
 ├── Services
 ├── Events
 └── Relations
```

## 3. Identity

用于唯一标识对象：

- ID
- 类型
- 名称
- 所属空间

## 4. Relations

描述对象之间关系：

- 连接关系
- 控制关系
- 空间关系
- 业务关系

## 5. Runtime映射

Thing实例加载后形成运行实体：

```
ThingDefinition
      ↓
ThingInstance
      ↓
RuntimeEntity
```

该设计支持模型复用和动态实例化。
