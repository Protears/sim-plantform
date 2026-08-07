# IW.SIM Part27 - Stacker Crane Simulation

## Overview

堆垛机模型用于自动化立库核心三维运动仿真。

## Coordinate System

支持：

- X Axis travel
- Y Axis lift
- Z Axis fork movement

## State Machine

状态包括：

- Idle
- Moving
- Loading
- Unloading
- Fault

## Control

支持 PLC/WCS 控制链路：

Command -> Runtime -> Motion Model -> Sensor/Event

## Simulation

支持速度、加速度、位置误差以及故障注入。