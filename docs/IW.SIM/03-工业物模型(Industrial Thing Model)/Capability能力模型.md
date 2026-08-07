# IW.SIM Capability能力模型设计

## 1. 概念

Capability描述工业对象能够执行的能力，是设备行为抽象的核心。

## 2. 为什么需要Capability

传统设备模型直接绑定品牌和协议，导致扩展困难。

IW.SIM通过能力抽象实现：

```
设备类型
   ↓
能力模型
   ↓
运行实现
```

## 3. 能力分类

### 运动能力

例如：

- 移动
- 升降
- 旋转

### 搬运能力

例如：

- 取货
- 放货
- 输送

### 控制能力

例如：

- 启停
- 速度控制
- 模式切换

## 4. 与PLC映射

Capability最终映射到控制接口：

```
Capability
 ↓
Command
 ↓
PLC IO
 ↓
Device Behavior
```

