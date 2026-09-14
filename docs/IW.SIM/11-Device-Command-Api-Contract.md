# Part 11：设备命令 API 与运行态查询契约

## 1. API 边界

API 只负责身份、参数校验、幂等键传递和 Application Service 调用，不直接调用 DeviceController。所有写操作都绑定 `RunId`，所有响应使用统一外层：`success/code/message/data/timestamp/details`。

## 2. 命令请求

```json
{
  "commandId": "uuid",
  "deviceId": "uuid",
  "cargoId": "uuid|null",
  "commandType": "Transfer",
  "requestHash": "sha256",
  "expectedDeviceVersion": 12,
  "targetSimTick": 8400,
  "deadlineSimTick": 9000,
  "payload": {}
}
```

`commandId` 由调用方生成并在重试中复用；服务端不得根据 HTTP RequestId 重新生成业务幂等键。

## 3. 端点

- `POST /api/v1/simulation-runs/{runId}/device-commands`
- `GET /api/v1/simulation-runs/{runId}/device-commands/{commandId}`
- `POST /api/v1/simulation-runs/{runId}/device-commands/{commandId}/cancel`
- `GET /api/v1/simulation-runs/{runId}/devices/{deviceId}/runtime-state`

返回状态：`202 Accepted` 表示已准入但未完成；`200` 表示查询成功；`409` 表示版本/幂等冲突；`422` 表示契约校验失败；`423` 表示资源被占用；`503` 表示运行时不可用。

## 4. 查询一致性

设备命令查询返回：`status`、`deviceVersion`、`transferId`、`occupancyVersion`、`lastEventSequence`。客户端不能把 `DeviceExecutionChanged` 当作位置提交完成；只有 `CargoTransferCommitted` 后才可显示目标位置已确认。

## 5. SignalR 推送

Hub：`/hubs/simulation-runs`。客户端按 RunId 加入组，推送事件至少包含 `eventId/runId/simTick/sequence/eventType/correlationId/schemaVersion`。客户端按 Sequence 去重并允许乱序缓存，直到连续序列补齐或收到重同步指令。

## 6. 安全与审计

命令写接口必须记录操作者、来源、RequestHash、原始 payload 摘要和结果码。Token 不进入事件 payload。Sealed Run 只允许查询和 Replay，不允许创建、取消或重试命令。

## 7. 验收

1. 重复 POST 返回同一 CommandId 的原始结果。
2. RequestHash 改变时返回 422，不覆盖原命令。
3. Cancel 与 Execute 竞争时状态转换符合 Part05 状态机。
4. SignalR 断线重连后可以通过 LastSequence 补发。
5. 设备完成与 Occupancy 提交在 UI 上明确区分。
