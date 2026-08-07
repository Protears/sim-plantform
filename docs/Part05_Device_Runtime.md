# Part05 Device Runtime

## 1. Overview

Device Runtime is the execution layer of IW.SIM responsible for transforming industrial equipment models into executable simulation entities. It bridges the Industrial Thing Model, World Model, Simulation Kernel and external control systems.

Device Runtime is not a simple equipment mock. It provides deterministic behavior simulation, lifecycle management, protocol adaptation, capability execution and runtime state synchronization.

## 2. Design Goals

### 2.1 Unified device execution model

Different industrial equipment such as conveyor, RGV, stacker crane, robot and elevator share a common runtime abstraction.

Core responsibilities:

- Device lifecycle management
- Command execution
- State machine transition
- Motion behavior simulation
- Sensor signal generation
- Fault injection
- Runtime telemetry collection

### 2.2 Separation of model and execution

The runtime does not contain hard-coded equipment knowledge.

```
Thing Model
    |
    v
Device Definition
    |
    v
Device Runtime Instance
    |
    v
Simulation Kernel Scheduler
```

The model describes capability; runtime executes capability.

## 3. Runtime Architecture

```
+------------------------------------------------+
|              Device Runtime Host               |
+------------------------------------------------+
| Device Lifecycle Manager                       |
| Capability Executor                            |
| Device State Machine                           |
| Motion Runtime                                 |
| IO Binding Runtime                             |
| Protocol Adapter                               |
| Event Publisher                                |
+------------------------------------------------+
              |
              v
+-----------------------------------------------+
| Simulation Kernel                             |
| SimTime / Event Queue / Deterministic Tick    |
+-----------------------------------------------+
```

## 4. Core Components

## 4.1 Device Runtime Manager

Responsible for runtime instance creation and destruction.

Functions:

- Load device definition
- Create runtime context
- Register simulation callbacks
- Manage runtime lifecycle

Lifecycle:

```
Created
  |
Initialized
  |
Ready
  |
Running
  |
Paused
  |
Stopped
  |
Disposed
```

## 4.2 Device Capability Executor

Capability is the executable contract exposed by a device.

Example:

```
Conveyor
 - Start()
 - Stop()
 - SetSpeed()
 - EmergencyStop()

Stacker Crane
 - MoveAxis()
 - Pickup()
 - Putdown()
```

Execution pipeline:

```
Command
  |
Validation
  |
Capability Resolver
  |
State Transition
  |
Simulation Event
  |
Result Event
```

## 4.3 Device State Machine

Every device owns an independent state machine.

Example conveyor:

```
Idle
 |
Start
 |
Running
 |
Blocked
 |
Fault
 |
Recover
```

State transition rules must be deterministic and replayable.

## 4.4 Motion Runtime

Motion Runtime provides common industrial movement behavior.

Supported models:

- Constant velocity
- Uniform acceleration
- Deceleration stopping
- Position tracking
- Collision boundary checking

Common parameters:

```
Position(m)
Velocity(m/s)
Acceleration(m/s²)
TargetPosition(m)
```

## 5. Device Communication Integration

Device Runtime exposes protocol independent interfaces.

```
                 Device Runtime
                       |
        +--------------+--------------+
        |                             |
     PLC Adapter                 API Adapter
        |                             |
     S7 / Modbus                 REST / SignalR
```

Communication layer only transfers commands and signals. Device behavior remains inside runtime.

## 6. Event Driven Execution

Device Runtime integrates with Simulation Kernel through events.

Examples:

```
CargoEnteredSensor
MotorStarted
PositionChanged
DeviceFaulted
TaskCompleted
```

Events contain:

- Simulation timestamp
- Device identity
- Event type
- Payload
- Trace information

## 7. Deterministic Simulation Requirements

Device Runtime must guarantee:

- Same input produces same output
- No uncontrolled wall-clock dependency
- All state changes occur through simulation time
- Replay support

## 8. Device Plugin Architecture

New equipment types are implemented as plugins.

Example:

```
IDeviceRuntime
    |
    +-- ConveyorRuntime
    +-- RgvRuntime
    +-- StackerCraneRuntime
    +-- RobotRuntime
```

Plugin responsibilities:

- Capability registration
- State definition
- Simulation behavior
- Signal mapping

## 9. Engineering Implementation (.NET)

Recommended structure:

```
IW.Sim.DeviceRuntime

/Application
/Domain
/Runtime
/Plugins
/Protocol
/Events
```

Key interfaces:

```csharp
public interface IDeviceRuntime
{
    string Id { get; }
    Task InitializeAsync();
    void Tick(SimTime time);
    Task ExecuteAsync(DeviceCommand command);
}
```

## 10. Future Extensions

- AI driven equipment behavior
- Physics based motion model
- ROS2 integration
- Unity/Unreal visualization synchronization
- Hardware-in-the-loop device bridge

Device Runtime is the execution foundation that enables IW.SIM to evolve from equipment simulation into a complete industrial cyber-physical simulation platform.
