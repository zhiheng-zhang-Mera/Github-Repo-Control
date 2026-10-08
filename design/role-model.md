# 角色与冲突模型

**GRC-MANAGER-V1.1 · 2026-10-08**

## 一、角色是什么

`CurrentMachine × Repository → Set<MachineRole>`，账户是管理会话，不改变同一物理主机的含义。平台能力、期望运行状态、GitHub 观测状态另存，不能都塞进角色。

**标签是路由声明，不是授权、执行隔离或验证独立性。** GRC 不禁止普通开发，不判定模型或任务级审查关系；同机从 Dev 切换成 Verify 不会抹除先前开发事实。历史角色变更写审计，但项目自行决定独立验证资格。

## 二、固定 Role Catalog

| Enum | 中文名 | GitHub label | 含义 |
|---|---|---|---|
| General | 一般任务 | `grc-role-general` | 普通 self-hosted 作业；不是“未注册”的别名 |
| CI | CI 测试 | `grc-role-ci` | 常规自动测试 |
| Build | 构建 | `grc-role-build` | 构建/打包 |
| Dev | 开发 | `grc-role-dev` | 声明为开发类作业用途 |
| Verify | 验证 | `grc-role-verify` | 声明为验证类用途，不认证独立性 |
| Repair | 修补 | `grc-role-repair` | 声明为修补类作业用途 |
| PlatformTest | 平台测试 | `grc-role-platform-test` | 当前真实平台上的专项测试 |

未注册的普通电脑用 NotRegistered 表示，不必勾 General。七项必须始终完整显示；所有可编辑角色来自 Core Catalog，不允许手工输入、Add custom role、raw label 编辑、命令行/配置隐式添加角色。未知 enum 拒绝。

PlatformTest 不代表拥有 Android/iOS SDK 或某型号 GPU。首版仅检查真实 OS/arch，显示能力描述；不探测或自动安装 SDK，也不伪造能力标签。

## 三、完整兼容矩阵

| A / B | General | CI | Build | Dev | Verify | Repair | PlatformTest |
|---|---|---|---|---|---|---|---|
| General | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |
| CI | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |
| Build | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |
| Dev | 允许 | 允许 | 允许 | 允许 | 冲突 | 允许 | 允许 |
| Verify | 允许 | 允许 | 允许 | 冲突 | 允许 | 冲突 | 允许 |
| Repair | 允许 | 允许 | 允许 | 允许 | 冲突 | 允许 | 允许 |
| PlatformTest | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |

唯一角色硬冲突为 Verify 与 Dev、Repair；不增加未获要求的新角色。保留 General+CI、CI+Build、CI+Verify 等组合。全部 128 个角色子集均可枚举测试：按纯角色冲突判定为 80 个兼容、48 个不兼容；运行状态/权限/安全门槛另测，不能混计。

## 四、编辑器行为

选择 Dev 或 Repair 后，Verify 立即 disabled 并说明原因；选择 Verify 后 Dev、Repair 同样禁用。取消冲突来源后恢复可选；不隐藏选项、不自动取消已选角色替用户决策。

每次编辑立即调用纯函数验证；Preview、Apply 再验证。远端有已知冲突组合时标 `InvalidRemoteRoleState`，与合法草案分区显示，允许用户重新选择合法组合修复；不得把冲突实况写成“合法选中项”。修复要先停止 Runner，再移除冲突旧标签，确认后添加新标签。

未知 `grc-role-*` 标 `UnsupportedPolicy`：显示但不可编辑或删除，不用旧版本规则猜测；仍允许本地停止。外部普通 custom labels 只读，不会成为可选角色。匹配大小写不敏感，输出标签使用目录中小写形式。

## 五、期望状态与不变量

| DesiredState | 本机要求 | 已知 GRC 标签 | 说明 |
|---|---|---|---|
| Active | 允许并请求运行；实际成功须双侧观测 | 与合法 DesiredRoles 一致且非空 | 不能只改 UI 就称已启用 |
| Standby | 确认停止 | 可保留此前已批准集合 | 不接单靠停止，不靠删标签 |
| Disabled | 确认停止 | 移除本版本已知 GRC 标签 | DesiredRoles 可作为下次草案保留 |
| Quarantined | 确认停止并保存安全锁 | 同 Disabled | 需用户解锁，不自动识别威胁 |

所有状态下保存的 DesiredRoles 都必须合法；不能因 Disabled 就保存 Dev+Verify。已知非法/未知远端实况单独记录。Active + 空集合拒绝；非 Active 可空。已解除锁不代表自动 Active。

执行顺序先处理安全锁与停止需求，再校验角色、能力、权限、归属及新鲜度；不是用某个状态跳过其它校验。紧急停止是独立减权操作，不要求远端角色合法或登录有效，但本机实例归属必须核实。

## 六、纯函数契约

定义位于 Core，签名及类型见 [shared-contracts.md](shared-contracts.md)：

- `RoleValidationResult Validate(IReadOnlySet<MachineRole> roles, DesiredState state, MachineCapabilities capabilities)`。
- `IReadOnlyList<RoleChoice> GetChoices(IReadOnlySet<MachineRole> currentRoles)`。
- Catalog 提供 id、中文名、说明、固定标签；RoleChoice 提供选中/可选/原因。

调用者不能偷偷规范化掉冲突选项让验证通过。传入集合天然去重；边界反序列化仍须拒绝未知值，不把数值溢出当 enum。

新增角色需更新目录、完整矩阵、子集测试及 UI 文案；没有新需求就不扩充。
