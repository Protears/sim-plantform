# IW.SIM Part26 - Conveyor Simulation Architecture

## Overview

输送机仿真是 IW.SIM 物流自动化仿真的核心设备模型。

## Model

Conveyor 由以下对象组成：

- Conveyor Line
- Segment
- Motor
- Sensor
- LoadUnit

## Motion

支持：

- Constant Speed Motion
- Uniform Accelerated Motion
- Stop / Resume
- Forward / Reverse

统一单位：

- speed: m/s
- acceleration: m/s²
- distance: m

## Sensor Model

Sensor 根据货载空间位置计算触发状态，支持：

- Presence Sensor
- Position Sensor
- Safety Sensor

## Runtime Integration

Conveyor Runtime 通过 Device Runtime 接入 Simulation Kernel，实现事件驱动仿真。