# Part 05：设备实例生命周期与健康状态契约

## 1. 生命周期

`Defined → Bound → Initializing → Ready → Running → Degraded → Recovering → Stopped → Retired`

禁止直接跳转：

- `Defined → Running`
- `Degraded → Ready`，必须经过 `Recovering` 或重新初始化。
- `Retired → 任意运行态`

## 2. C# 接口

```csharp
public interface IDeviceHealthCoordinator
{
    ValueTask<DeviceHealthSnapshot> GetAsync(
        DeviceInstanceId deviceId, CancellationToken ct);

    ValueTask<HealthTransitionResult> TransitionAsync(
        DeviceHealthTransition transition, CancellationToken ct);
}
```

## 3. 健康判定维度

- 配置：Definition/MappingHash 是否有效。
- 通信：当前 Epoch、最近有效输入、Ack 延迟。
- 执行：命令队列深度、执行超时、锁租约。
- 位置：OccupancyVersion 与设备内部位置状态是否一致。
- 反馈：ProcessImageVersion 是否单调。
- 安全：Critical Quality、FailSafe 和急停状态。

## 4. 降级规则

- 通信超时：进入 `Degraded`，禁止新 Critical Command，允许安全停止。
- Occupancy 对账失败：进入 `Recovering`，禁止 CargoTransferCompleted。
- 锁租约丢失：立即释放可释放资源并进入 `Recovering`。
- 配置摘要不一致：阻断 `Ready`。

## 5. 事件

- `DeviceHealthChanged`
- `DeviceRecoveryStarted`
- `DeviceRecoveryCompleted`
- `DeviceSafetyBlocked`

所有事件包含 `RunId、DeviceId、DeviceVersion、SimTick、CorrelationId、SchemaVersion`。

## 6. 验收

- 任何设备不得在未 Ready 时执行普通命令。
- Degraded 设备不能生成新的 Critical Output。
- Recovery 未完成时不得发布 Completed 类事件。
