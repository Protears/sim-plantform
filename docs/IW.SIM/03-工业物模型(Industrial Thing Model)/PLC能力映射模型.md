# IW.SIM PLC能力映射模型

## 1. 概述

PLC能力映射模型用于连接工业物模型与实际控制系统，实现仿真设备与PLC程序之间的数据交互。

## 2. 映射关系

```text
Thing Capability
        ↓
PLC Signal
        ↓
IO Address
        ↓
Runtime Adapter
```

## 3. 映射内容

包括：

- 输入信号
- 输出信号
- 状态反馈
- 控制命令
- 报警信息

## 4. 设计目标

实现：

- PLC程序无需修改即可连接仿真环境；
- 支持Siemens S7等工业协议；
- 支持虚拟调试闭环验证。
