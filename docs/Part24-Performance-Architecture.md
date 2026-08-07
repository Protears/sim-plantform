# IW.SIM Part24 - Performance Architecture

## Goals

保证大规模工业场景稳定运行。

## Design

- Tick Optimization
- Event Queue Optimization
- Object Pool
- Async Pipeline
- Cache Strategy

## Targets

- Tick cycle 30ms
- Hundreds of devices simulation
- Long-time stable execution

## Monitoring

结合 OpenTelemetry 监控性能指标。