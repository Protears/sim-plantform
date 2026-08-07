# IW.SIM Part23 - Event Driven Architecture

## Overview

事件驱动架构用于连接 Simulation Kernel、Runtime 和应用层。

## Event Flow

```text
Entity Change
 -> Domain Event
 -> Event Bus
 -> Runtime/Application
```

## Components

- Event Dispatcher
- Event Queue
- Event Store
- Subscription Service

支持实时仿真、回放和审计。