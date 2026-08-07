# IW.SIM Design Documentation

Industrial World Simulation Platform (IW.SIM) architecture documentation.

## Document Structure

- 01-Overview.md - Platform overview and vision
- 02-World-Model.md - World model and industrial object model
- 03-Architecture.md - Overall software architecture
- 04-Simulation-Engine.md - Discrete event simulation engine design
- 05-Device-Model.md - Device modeling and runtime design
- 06-PLC-Control-Simulation.md - PLC and control simulation
- 07-Digital-Twin.md - Digital twin and HIL integration
- 08-Engineering.md - Engineering platform design

## Technology Baseline

- .NET 10 / C# 13
- ASP.NET Core 10
- EF Core 10
- PostgreSQL
- Vue 3 + TypeScript
- PixiJS visualization
- OpenTelemetry observability

## Goal

Build an industrial mechatronics integrated simulation platform supporting equipment simulation, PLC integration, WCS/WMS validation, digital twin and hardware-in-loop scenarios.
