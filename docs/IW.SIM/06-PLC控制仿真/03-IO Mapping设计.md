# IO Mapping设计

## 1. 概述

IO Mapping负责连接设备模型信号、PLC地址以及工程控制点位。

## 2. 映射层次

```text
Device Signal
      ↓
Capability Signal
      ↓
IO Mapping
      ↓
PLC Address
```

## 3. 支持类型

- 输入信号
- 输出信号
- 状态反馈
- 报警信号
- 命令信号

## 4. 设计目标

支持 Excel 导入、项目配置以及运行时动态绑定。