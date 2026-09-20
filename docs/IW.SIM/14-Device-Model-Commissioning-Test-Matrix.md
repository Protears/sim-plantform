# Part 14：设备模型装配与联调验收矩阵

## 1. 目标

验证 Definition、Configuration、Capability、PointMapping、Runtime、PLC Feedback、Occupancy 和 Recovery 在设备实例装配阶段形成闭环。

| ID | 场景 | 前置条件 | 期望结果 |
|---|---|---|---|
| DM-001 | 正常绑定 | Definition/Config/Mapping 合法 | 生成 BindingSnapshot，实例进入 Bound |
| DM-002 | 缺少必需点位 | Capability 要求 point-entry | 返回 POINT-MAP-404，不进入 Ready |
| DM-003 | 地址重复冲突 | 两语义映射同一地址 | 返回 POINT-MAP-409 |
| DM-004 | MappingHash 变化 | 旧 Snapshot 已存在 | 生成新 Snapshot，旧 Run 不复用 |
| DM-005 | 配置冻结修改 | Run 已 Frozen | 返回 CONFIG-FROZEN-409，产生审计 |
| DM-006 | Bad Critical Quality | 输入 Quality=Bad | 阻止 Critical Command |
| DM-007 | 设备未 Ready 下命令 | 状态=Bound | 命令拒绝，不改变设备版本 |
| DM-008 | 多 Cargo 并发 | 两命令竞争同一 Cargo | 仅一个 Transfer 成功 |
| DM-009 | Occupancy 对账失败 | StateHash 不一致 | 设备进入 Recovering，不发布 Completed |
| DM-010 | 断线恢复 | Epoch 递增 | 旧 Epoch 输入拒绝，新 Epoch 继续 |
| DM-011 | Snapshot 恢复 | Active Transfer 存在 | 恢复后不重复 Commit |
| DM-012 | Replay | Frozen BindingSnapshot | Replay Hash 与原运行一致 |

## 2. 质量门禁

- DM-001～DM-006：配置与映射门禁。
- DM-007～DM-009：运行时安全门禁。
- DM-010～DM-012：恢复与确定性门禁。
- 任一 P0 场景失败，不得进入 PLC/HIL 联调。

## 3. 测试证据

每个测试必须保留：RunId、DeviceInstanceId、DefinitionVersion、MappingHash、CommandId/TransferId、EventSequence、错误码、最终 DeviceVersion、OccupancyVersion 和 StateHash。
