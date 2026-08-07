# IW.SIM Part03 - Simulation Kernel

## 1. Overview

Simulation Kernel 是 IW.SIM 平台的仿真执行核心，负责工业世界状态推进、事件调度、时间管理和确定性执行。

核心目标：

- 提供高性能离散事件仿真能力
- 支持实时与非实时混合仿真
- 保证仿真结果可重复
- 支持调试、回放和故障分析

## 2. Architecture

```text
Simulation Kernel
│
├── SimTime
├── Event Scheduler
├── Event Queue
├── Entity Runtime
├── State Manager
├── Determinism Policy
└── Replay Engine
```

## 3. Simulation Time Model

IW.SIM 使用独立仿真时间 SimTime，不直接依赖操作系统时间。

```text
SimTime
 ├── Current Tick
 ├── Simulation Timestamp
 └── Time Scale
```

支持：

- Real Time
- Fast Forward
- Step Execution
- Pause / Resume

## 4. Discrete Event Simulation

系统采用事件驱动模型：

```text
Event
 ├── Timestamp
 ├── Priority
 ├── Source
 ├── Payload
 └── Handler
```

事件按照时间顺序进入 Event Queue，由 Scheduler 推进执行。

## 5. Event Scheduler

Scheduler 负责：

- 事件排序
- 时间推进
- 生命周期管理
- Tick 调度

典型流程：

```text
while running
{
    nextEvent = Queue.Pop()
    SimTime.Advance(nextEvent.Time)
    Execute(nextEvent)
    PublishStateChange()
}
```

## 6. Real-time Hybrid Simulation

支持两种运行模式：

### Accelerated Simulation

用于：

- 方案验证
- 算法测试
- 自动化测试

### Real-time Simulation

用于：

- PLC 联调
- WCS 联调
- HIL 测试

## 7. Tick Architecture

标准运行周期：

```text
30ms Tick

Input Update
      ↓
Device Runtime
      ↓
Physics / Motion Update
      ↓
Event Dispatch
      ↓
Output Synchronization
```

设计目标：

- 时间漂移 ≤60ms
- 支持100+设备并行仿真

## 8. Deterministic Execution

确定性策略保证：

同一输入 + 同一随机种子 = 同一仿真结果

实现机制：

- 固定随机种子
- 有序事件队列
- 禁止非确定性共享状态
- Event Log 记录

## 9. Replay and Debug

Kernel 支持运行记录：

```text
Simulation Snapshot
        +
Event Log
        ↓
Replay Engine
```

应用场景：

- Bug定位
- 自动测试失败复现
- 现场问题分析

## 10. Runtime Integration

Simulation Kernel 与其他模块关系：

```text
World Model
      ↓
Simulation Kernel
      ↓
Device Runtime
      ↓
PLC Runtime
      ↓
Application
```

## 11. Implementation Direction

推荐技术实现：

- C# / .NET 10
- 高性能 Priority Queue
- Channel/Event Pipeline
- MemoryPool
- OpenTelemetry Metrics

Simulation Kernel 是 IW.SIM 实现工业级数字化仿真和机电控联调能力的核心基础。