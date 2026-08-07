# IW.SIM 确定性仿真设计

## 1. 设计目标

确定性仿真（Deterministic Simulation）是 IW.SIM 仿真内核的重要基础能力，用于保证相同输入条件下，仿真结果保持一致。

该能力支撑：

- 自动化测试；
- 故障复现；
- 仿真回放；
- 算法验证；
- 虚拟调试。

## 2. 核心原则

### 时间确定性

所有设备行为必须基于统一 SimClock，而不是依赖操作系统时间。

### 事件确定性

事件按照统一排序规则执行：

1. 时间戳；
2. 优先级；
3. 注册顺序。

### 状态确定性

每次状态变化必须通过 Runtime 管道产生。

## 3. 执行模型

```
Simulation Time
        ↓
Event Queue
        ↓
Behavior Execute
        ↓
State Transition
        ↓
Observation
```

## 4. 应用场景

- PLC程序回归测试；
- WCS策略验证；
- 现场问题复现；
- 自动化测试流水线。
