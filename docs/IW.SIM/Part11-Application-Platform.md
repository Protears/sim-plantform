# IW.SIM Part11 - Application Platform

## 1. 文档定位

Application Platform 是 IW.SIM 面向用户、项目、场景、仿真运行、联调、测试与运维的应用平台层。

本层不重复定义 World Model、Simulation Kernel、Device Runtime、PLC Runtime、Signal/IO Runtime、Communication 与 Simulation Data 的内部实现，而是把这些能力组织成可操作、可配置、可观测、可验证的产品能力。

Application Platform 的核心目标不是“提供一个 Web UI”，而是建立完整的仿真工程生命周期：

```text
项目创建
  -> 场景建模
  -> 模型校验
  -> 运行配置
  -> 仿真启动
  -> WCS/PLC/设备联调
  -> 运行观测
  -> 测试执行
  -> 结果分析
  -> 缺陷定位
  -> 场景复现
  -> 结果归档
```

平台必须支持“配置数据、运行时状态、控制命令、仿真事件、测试结果”五类信息分离，避免前端直接操作 Runtime 内部状态。

---

## 2. 分层架构

```text
+------------------------------------------------------------+
|                    IW.SIM Application                      |
+------------------------------------------------------------+
| Project | Scene | Runtime | Integration | Test | Monitor |
+------------------------------------------------------------+
| Application API / Command / Query / SignalR                |
+------------------------------------------------------------+
| Application Services                                       |
| ProjectService | SceneService | RunService | TestService  |
+------------------------------------------------------------+
| Domain / Simulation Facades                                |
| World | Simulation | Runtime | Communication | Data       |
+------------------------------------------------------------+
| Simulation Platform Core                                   |
| World Model | Kernel | Device Runtime | PLC | IO | Event  |
+------------------------------------------------------------+
| Infrastructure                                             |
| PostgreSQL | Cache | Event Store | File/Object Storage     |
+------------------------------------------------------------+
```

### 2.1 Application Layer职责

Application Layer 负责：

- 用例编排；
- 权限边界；
- 项目上下文；
- 场景生命周期；
- 仿真运行生命周期；
- 外部系统连接配置；
- 测试任务编排；
- 查询与结果聚合；
- 操作审计；
- API 与 SignalR 对外暴露。

Application Layer 不负责：

- 设备物理运动计算；
- PLC 扫描执行；
- IO 信号底层处理；
- Event Scheduler 内部调度；
- 数据库 ORM 细节；
- 三维渲染细节。

---

## 3. 核心领域上下文

Application Platform 定义以下业务上下文：

|上下文|职责|核心对象|
|-|-|-|
|Project|工程隔离与生命周期|Project、ProjectVersion|
|Scene|仿真场景管理|Scene、SceneVersion、ModelReference|
|Run|仿真运行管理|SimulationRun、RunConfiguration|
|Integration|外部系统联调|Endpoint、ConnectionProfile、Binding|
|Test|自动化验证|TestSuite、TestCase、TestRun、Assertion|
|Observation|运行观测|View、Query、Subscription|
|Result|实验结果|RunResult、Metric、Artifact|
|Administration|平台管理|User、Role、AuditRecord|

这些上下文通过 Application Service 协作，不直接共享数据库实体。

---

## 4. Project 工程模型

Project 是用户管理 IW.SIM 工程的顶层边界。

```text
Project
 ├── ProjectMetadata
 ├── ProjectVersion
 │    ├── SceneVersion
 │    ├── DeviceModels
 │    ├── IOMappings
 │    ├── PLCProfiles
 │    ├── IntegrationProfiles
 │    └── RunProfiles
 ├── TestSuites
 └── Artifacts
```

### 4.1 Project 状态机

```text
Draft
  |
Editing
  |
Validated
  |
Published
  |
Archived
```

状态约束：

- Draft：允许编辑；
- Editing：允许创建版本和修改模型；
- Validated：通过结构及引用校验；
- Published：作为可运行版本，不允许直接修改；
- Archived：只读，用于历史追溯。

任何 SimulationRun 必须绑定一个不可变的 ProjectVersion，不能直接绑定可编辑 Project。

---

## 5. Version 管理

IW.SIM 必须采用版本化模型避免“运行中的场景被修改导致无法复现”。

```text
Project
   |
   +-- Version 1
   +-- Version 2
   +-- Version 3
             |
             +-- Simulation Run 301
             +-- Simulation Run 302
```

### 5.1 版本不可变原则

一旦 Version 被 Run 引用：

- 不允许覆盖原始配置；
- 不允许改变模型身份；
- 不允许修改 IO Mapping；
- 不允许修改 Runtime Profile；
- 不允许改变外部连接语义。

新的设计必须生成新的 Version。

这样才能保证：

> Run Result + ProjectVersion + Input + Seed 可以重建同一实验条件。

---

## 6. Scene 管理

Scene 是可运行的工业世界配置。

```text
Scene
 ├── SpatialModel
 ├── WorldEntities
 ├── Devices
 ├── LoadUnits
 ├── Sensors
 ├── Tracks
 ├── Connections
 ├── PLC Mapping
 └── Runtime Configuration
```

### 6.1 Scene 校验

发布前必须执行静态校验：

1. Entity ID 唯一性；
2. 引用完整性；
3. 空间拓扑合法性；
4. Device Capability 完整性；
5. Signal/IO Mapping 完整性；
6. PLC 地址合法性；
7. Runtime Profile 存在；
8. Communication Endpoint 可解析；
9. 不允许循环或非法拓扑关系；
10. 必要资源是否存在。

校验结果分为：

```text
Error   -> 禁止发布
Warning -> 允许发布但记录风险
Info    -> 提供诊断信息
```

---

## 7. Simulation Run 模型

SimulationRun 是一次实际执行的仿真实验实例。

```text
SimulationRun
{
    Id,
    ProjectId,
    ProjectVersionId,
    SceneVersionId,
    RunProfileId,
    Mode,
    SimTime,
    Status,
    Seed,
    StartedAt,
    CompletedAt,
    FailureReason
}
```

### 7.1 Run 生命周期

```text
Created
  |
Preparing
  |
Ready
  |
Running <------+
  |             |
  +--> Paused --+
  |
  +--> Stopping
  |
Completed / Failed / Cancelled
```

### 7.2 Run 状态转换约束

- Created -> Preparing；
- Preparing -> Ready / Failed；
- Ready -> Running；
- Running -> Paused / Stopping / Failed；
- Paused -> Running / Stopping；
- Stopping -> Completed / Failed / Cancelled；
- Completed、Failed、Cancelled 为终态。

终态 Run 不允许再次启动；需要重新执行时创建新的 Run。

---

## 8. Run Profile

RunProfile 将运行参数与项目模型分离。

核心参数：

```yaml
clock:
  mode: realtime
  speedRatio: 1.0
  tickInterval: 30ms

random:
  seed: 20260908

runtime:
  deterministic: true
  parallelism: 1

integration:
  wcs: enabled
  plc: enabled
  externalDevices: disabled

observation:
  eventLevel: normal
  stateSampling: 100ms
```

### 8.1 确定性要求

对于 deterministic Run：

- Seed 固定；
- Event Scheduler 排序规则固定；
- 同一 SimTime 内事件具有确定优先级；
- 并行执行不能改变可观察结果；
- 外部输入必须记录；
- 非确定性外部数据必须进入 Replay Input。

---

## 9. Application Command 模型

所有会改变系统状态的操作使用 Command，而不是通用 HTTP Update。

典型 Command：

```text
CreateProject
PublishProjectVersion
ValidateScene
CreateSimulationRun
StartSimulation
PauseSimulation
ResumeSimulation
StopSimulation
ResetSimulation
AttachExternalController
DetachExternalController
StartTestRun
CancelTestRun
CreateSnapshot
RestoreSnapshot
```

Command 必须具备：

- CommandId；
- ActorId；
- CorrelationId；
- TargetId；
- ExpectedVersion；
- IssuedAt；
- Payload。

### 9.1 幂等

Command 必须支持幂等处理。

```text
CommandId
   |
Idempotency Store
   |
+-- Already Executed -> Return Previous Result
|
+-- New -> Execute -> Persist Result
```

特别是 Start、Stop、Reset、TestRun 等操作，重复请求不能产生重复 Runtime 实例。

---

## 10. Query 模型

查询采用 Query Service，避免将 Domain Entity 直接暴露给 API。

```text
IProjectQueryService
ISceneQueryService
ISimulationRunQueryService
IDeviceQueryService
ITestQueryService
IResultQueryService
```

查询返回专用 Read Model：

```csharp
public sealed record SimulationRunSummary(
    Guid RunId,
    string ProjectVersion,
    string Status,
    long SimTick,
    double SimTime,
    DateTimeOffset? StartedAt,
    DateTimeOffset? CompletedAt);
```

Read Model 可以针对页面查询进行反范式化，不反向污染领域模型。

---

## 11. API 设计

外部 API 采用版本化 REST：

```text
/api/v1/projects
/api/v1/scenes
/api/v1/simulation-runs
/api/v1/test-suites
/api/v1/results
```

写操作采用 Command Endpoint：

```http
POST /api/v1/simulation-runs/{runId}/commands/start
POST /api/v1/simulation-runs/{runId}/commands/pause
POST /api/v1/simulation-runs/{runId}/commands/resume
POST /api/v1/simulation-runs/{runId}/commands/stop
```

查询采用资源 Endpoint：

```http
GET /api/v1/projects/{projectId}
GET /api/v1/scenes/{sceneId}
GET /api/v1/simulation-runs/{runId}
GET /api/v1/simulation-runs/{runId}/events
GET /api/v1/simulation-runs/{runId}/metrics
```

API 不直接返回数据库实体。

---

## 12. SignalR 实时通道

SignalR 用于运行态推送，而不是替代 REST。

```text
REST
  -> Command / Query

SignalR
  -> Runtime Event / State / Progress
```

Hub 分组建议：

```text
project:{projectId}
run:{runId}
test-run:{testRunId}
```

运行期间只向订阅者推送必要数据，不能把完整 World State 在每个 Tick 广播给所有客户端。

事件推送示例：

```json
{
  "type": "simulation.state.changed",
  "runId": "run-001",
  "simTime": 12.430,
  "sequence": 18321,
  "changes": [
    {
      "entityId": "CV001",
      "path": "runtime.speed",
      "value": 0.8
    }
  ]
}
```

Sequence 用于客户端检测丢包和乱序。

---

## 13. External Integration 管理

Application Platform 不直接实现 Siemens S7、OPC UA、MQTT 等协议，而管理 Integration Profile。

```text
IntegrationProfile
 ├── Protocol
 ├── Endpoint
 ├── AuthenticationReference
 ├── DeviceBindings
 ├── SignalMappings
 ├── TimeoutPolicy
 ├── RetryPolicy
 └── HealthCheckPolicy
```

真实凭据不得进入 ProjectVersion；仅保存 Credential Reference。

Integration Connection 生命周期：

```text
Disconnected
 -> Connecting
 -> Connected
 -> Degraded
 -> Reconnecting
 -> Disconnected
```

异常必须可观测并进入 Run Diagnostic。

---

## 14. 自动化测试平台

Application Platform 必须提供 Simulation Test Harness，使仿真成为可自动执行的测试环境。

```text
TestSuite
  |
  +-- TestCase
       |
       +-- Setup
       +-- Input
       +-- Stimulus
       +-- Observation
       +-- Assertion
       +-- Cleanup
```

### 14.1 TestCase

```yaml
name: ConveyorCargoTransfer
setup:
  scene: conveyor-demo
stimulus:
  - command: createCargo
    position: 0.2
  - wait: 5s
assertions:
  - path: cargo.currentLocation
    equals: CV002
  - path: sensor.S10102.active
    equals: true
```

测试断言必须支持：

- 状态断言；
- 事件断言；
- 时间断言；
- 数值范围；
- 最终状态；
- 禁止事件；
- 顺序约束。

---

## 15. Test Run 与仿真 Run 解耦

TestRun 不等于 SimulationRun。

```text
TestRun
   |
   +-- SimulationRun 001
   +-- SimulationRun 002
   +-- SimulationRun 003
```

同一个 TestCase 可以重复执行，并产生不同 SimulationRun。

TestRun 保存：

- TestCase Version；
- ProjectVersion；
- RunProfile；
- Input Set；
- Seed；
- SimulationRunId；
- Assertion Results；
- Artifact References。

这样能够实现回归测试和失败复现。

---

## 16. 观测与诊断

Application Platform 负责将底层可观测能力聚合成用户可理解的 Diagnostic。

诊断维度：

```text
Run
 |
 +-- Runtime State
 +-- Simulation Events
 +-- Communication
 +-- PLC / IO
 +-- Device
 +-- Performance
 +-- Errors
```

每个故障必须尽可能关联：

```text
Error
 -> RunId
 -> SimTime
 -> EntityId
 -> CorrelationId
 -> EventId
 -> TraceId
```

避免出现“页面显示设备异常，但无法定位到具体事件和控制链路”的黑盒问题。

---

## 17. 权限模型

权限采用 RBAC，并以 Project 为主要资源边界。

```text
User
  |
Role
  |
Permission
  |
Resource Scope
```

典型权限：

- project.read
- project.write
- project.publish
- scene.edit
- simulation.start
- simulation.control
- integration.manage
- test.execute
- result.read
- administration.manage

运行控制必须额外检查 Project 权限和 Run 所属 Version 的访问权限。

---

## 18. 审计设计

所有重要写操作必须产生 AuditRecord：

```text
AuditRecord
{
 Id,
 ActorId,
 ProjectId,
 Action,
 TargetType,
 TargetId,
 CorrelationId,
 Before,
 After,
 OccurredAt
}
```

至少审计：

- Project 发布；
- Scene 修改；
- Integration 配置变更；
- Simulation Start/Stop/Reset；
- Test 执行；
- Snapshot Restore；
- 权限变化。

Audit Log 与 Simulation Event 不混合。前者描述“谁做了什么”，后者描述“仿真世界发生了什么”。

---

## 19. 前端架构

Vue 3 + TypeScript 前端采用 Feature-oriented 结构：

```text
src/
  modules/
    project/
    scene/
    simulation/
    integration/
    testing/
    monitoring/
    result/
  shared/
    api/
    signalr/
    components/
    models/
```

PixiJS 负责高频场景渲染，不承担领域状态管理。

前端状态分为：

- Server State：由 API 管理；
- Runtime Stream：由 SignalR 管理；
- UI State：由前端管理；
- Scene Editing State：本地编辑模型，发布时提交服务端。

禁止把完整 Simulation Runtime State 作为 Vue 全局响应式对象持续深拷贝。

---

## 20. 性能设计

Application Platform 不应成为 Simulation Kernel 的实时执行瓶颈。

原则：

1. Simulation Runtime 与 Web Request 生命周期解耦；
2. API 不阻塞 Simulation Tick；
3. SignalR 推送采用增量状态；
4. 大量历史数据采用分页/聚合查询；
5. 前端渲染与仿真 Tick 解耦；
6. 长耗时操作采用 Job/Run 模式；
7. 查询不得直接扫描 Event Store 全量数据。

目标约束：

- 常规控制命令不应长时间占用 HTTP 请求；
- 运行态推送应支持背压；
- 单个客户端不能因订阅大量实体拖垮 Runtime；
- 大型场景必须支持按空间、实体类型和变更集进行增量查询。

---

## 21. 并发与一致性

Application Command 与 Simulation Runtime 之间采用版本检查：

```text
Client Command
     |
ExpectedVersion
     |
Command Handler
     |
Runtime State Version Check
     |
Execute / Reject
```

并发控制重点：

- 防止重复 Start；
- 防止 Stop 与 Reset 竞争；
- 防止两个客户端同时发布同一 Project；
- 防止旧页面覆盖新版本；
- 防止 TestRun 与人工控制互相干扰。

对于不可并行操作，使用资源级 Command Lock，而不是数据库全局锁。

---

## 22. 故障恢复

### 22.1 Application Service 故障

Application API 重启不应导致 Simulation Runtime 必然丢失。

Runtime 与 Application Host 解耦后：

```text
Application API
      |
Runtime Manager
      |
Simulation Host
```

运行状态通过 Run Registry 和 Runtime Heartbeat 管理。

### 22.2 Runtime 故障

Runtime 异常后根据策略：

```text
Detect
  |
Freeze
  |
Capture Diagnostic
  |
Persist Failure
  |
Restore Snapshot (optional)
  |
Resume / Restart / Abort
```

恢复必须明确区分：

- 从 Snapshot 恢复；
- 从头重新运行；
- 从 Replay Input 重演。

不能把三者混为“重启”。

---

## 23. 数据访问边界

Application Service -> Domain/Simulation Facade -> Data Service -> Repository/Store。

禁止：

```text
Controller -> DbContext
Controller -> Simulation Internal State
Frontend -> Database
Frontend -> Runtime Internal API
```

这样可以保持 Clean Architecture / DDD 的边界，并允许后续替换数据存储或 Runtime Hosting 模式。

---

## 24. .NET 工程映射

推荐解决方案结构：

```text
IW.SIM.Application.sln
 |
 +-- IW.SIM.Domain
 +-- IW.SIM.Application
 +-- IW.SIM.Infrastructure
 +-- IW.SIM.Runtime
 +-- IW.SIM.Integration
 +-- IW.SIM.Web.Entry
 +-- IW.SIM.Contracts
 +-- IW.SIM.Tests
```

### Application 项目建议

```text
Application/
  Projects/
    Commands/
    Queries/
    Handlers/
  Scenes/
  SimulationRuns/
  Testing/
  Integration/
  Results/
  Authorization/
```

Application Handler 只编排用例，不承载设备运动算法。

---

## 25. 典型启动流程

```text
POST CreateSimulationRun
        |
        v
Validate ProjectVersion
        |
        v
Validate SceneVersion
        |
        v
Resolve RunProfile
        |
        v
Allocate Runtime
        |
        v
Load World Model
        |
        v
Initialize Device Runtime
        |
        v
Initialize PLC / IO / Communication
        |
        v
Create SimulationRun
        |
        v
Ready
        |
        v
StartSimulation
```

任何初始化失败都必须产生结构化 FailureReason，并释放已经分配的资源。

---

## 26. 典型停止流程

```text
Stop Command
    |
Validate State
    |
Freeze New Commands
    |
Stop External Integration
    |
Stop Runtime Scheduling
    |
Flush Event / Result
    |
Persist Final State
    |
Release Runtime
    |
Completed
```

停止流程必须具备超时策略，不能因为一个外部设备连接未关闭而永久阻塞整个 Run。

---

## 27. 关键架构决策

### ADR-APP-001：ProjectVersion 不可变

**Decision:** SimulationRun 必须引用不可变 ProjectVersion。

**原因:** 保证实验结果可复现，避免运行过程中配置漂移。

### ADR-APP-002：REST + SignalR 双通道

**Decision:** Command/Query 使用 REST，运行态事件使用 SignalR。

**原因:** 控制面与实时数据面职责不同，避免 HTTP 长轮询和高频状态查询。

### ADR-APP-003：TestRun 与 SimulationRun 分离

**Decision:** 自动化测试通过 TestRun 编排 SimulationRun。

**原因:** 一个测试需要支持多次执行、失败重试、不同 Seed 和不同 RunProfile。

### ADR-APP-004：Application 不持有设备算法

**Decision:** Application Layer 只能调用 Runtime/Domain Facade。

**原因:** 防止产品应用层逐渐侵入仿真核心，导致模块边界失控。

---

## 28. 验收标准

Application Platform 设计完成必须满足：

### 工程管理

- [ ] Project 可创建、复制、版本化、发布、归档；
- [ ] Published Version 不可变；
- [ ] Run 始终绑定明确 Version。

### 仿真运行

- [ ] Run 生命周期完整；
- [ ] Start/Pause/Resume/Stop/Reset 状态约束明确；
- [ ] Command 幂等；
- [ ] Runtime 与 Web 请求解耦。

### 联调

- [ ] PLC/WCS/设备连接均通过 Integration Profile 管理；
- [ ] 外部连接故障可恢复；
- [ ] 凭据不进入版本配置。

### 测试

- [ ] TestSuite/TestCase/TestRun 模型完整；
- [ ] 支持状态、事件、时间和数值断言；
- [ ] 测试结果可追溯到 ProjectVersion、RunProfile、Seed。

### 可观测性

- [ ] Run、Event、Entity、CorrelationId、TraceId 可以关联；
- [ ] 操作审计与仿真事件分离；
- [ ] 失败可以定位到运行链路。

### 可扩展性

- [ ] Application 不依赖具体设备品牌；
- [ ] Runtime 不依赖 Web；
- [ ] API 支持版本化；
- [ ] 前端不依赖 Runtime 内部对象；
- [ ] 新增设备类型不需要修改 Application 核心流程。

---

## 29. 与其他章节的关系

```text
Part01 Architecture
        |
        v
Part02 World Model
        |
        +--> Part04 Thing Model
        |
        v
Part03 Simulation Kernel
        |
        +--> Part05 Device Runtime
        +--> Part06 PLC Runtime
        +--> Part07 Signal / IO Runtime
        +--> Part08 Communication
        |
        v
Part09 Digital Twin Integration
        |
        v
Part10 Simulation Data
        |
        v
Part11 Application Platform
        |
        +--> Test / Monitor / Result
        |
        v
Part12 HIL
        |
        v
Part13 Observability
        |
        v
Part14 Engineering Implementation
```

Application Platform 是核心仿真能力向工程产品能力的承接层。它不重新定义底层运行时，而是通过明确的 Application Service、Command、Query、Run、Test 和 Integration 边界，把底层能力组织成可交付的平台。

---

# 30. 总结

IW.SIM Application Platform 的核心不是页面，而是完整的工业仿真工程生命周期管理能力。

通过 Project Version、Scene Version、Simulation Run、Integration Profile、Test Run、Result 和 Audit 等核心模型，将：

```text
模型
  -> 配置
  -> 校验
  -> 运行
  -> 联调
  -> 测试
  -> 观测
  -> 分析
  -> 复现
```

形成闭环。

该层最终应使 IW.SIM 从“能够运行仿真引擎”提升为“能够承载工业自动化项目开发、联调、验证和交付”的工程平台，同时保持 Application、Runtime、World Model、Data 与 Integration 的架构边界。