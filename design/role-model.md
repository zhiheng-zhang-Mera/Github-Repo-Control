# 角色与冲突模型

## 1. 基本原则

Role 是：

```text
CurrentMachine × Repository → Set<Role>
```

不是：

```text
CurrentMachine → OneRole
```

因此同一台机器可以：

```text
Repo A: general + ci
Repo B: build
Repo C: verify
```

## 2. 首版角色

角色集合由应用内置的 Role Catalog 提供。**用户不得手工输入角色名、自定义字符串或直接编辑 `grc-role-*` label。**

Repository Detail 必须把当前版本所有可选角色完整展示为 checkbox/toggle/list item。未来新增角色时，由版本升级扩充 Role Catalog，而不是给用户提供自由文本输入框。

| Role | Label | 含义 |
|---|---|---|
| General | `grc-role-general` | 普通 self-hosted 工作节点 |
| CI | `grc-role-ci` | CI/自动测试 |
| Build | `grc-role-build` | 构建/打包 |
| Dev | `grc-role-dev` | 开发/施工 |
| Verify | `grc-role-verify` | 独立验证 |
| Repair | `grc-role-repair` | 修补 |
| Platform Test | `grc-role-platform-test` | 当前平台专项测试 |

## 3. 状态不是角色

以下使用单独 AssignmentState，不进入多选 Role：

- Active
- Standby
- Disabled
- Quarantined

规则：

- Active：按角色接任务。
- Standby：保留注册，但不应接新任务。
- Disabled：本 repo 的当前主机 runner 停用。
- Quarantined：安全隔离；优先级最高。

非 Active 状态不得通过偷偷保留 active role label 来继续接单。

## 4. 兼容矩阵

符号：

- ✅ 允许
- ⚠️ 允许但 UI 提示
- ❌ 硬冲突

| A \ B | General | CI | Build | Dev | Verify | Repair | Platform |
|---|---:|---:|---:|---:|---:|---:|---:|
| General | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| CI | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Build | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Dev | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| Verify | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Repair | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| Platform | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### 为什么 Verify 与 Dev/Repair 冲突

这里的 Verify 定义为“独立验证角色”。同一 repo 上同一主机如果同时声明开发/修补与独立验证，会破坏角色语义，因此硬拒绝。

如果未来需要“普通测试但不要求独立性”，应使用 CI / Platform Test，而不是降低 Verify 的约束。

## 5. 规则优先级

```text
Quarantined
  > Disabled
  > Standby
  > Hard Conflict
  > Capability Constraint
  > Warning
  > Allowed
```

只要命中更高优先级规则，后面的允许项不能覆盖它。

## 6. 角色选择器行为

UI 中的角色集合在任何时刻都必须保持合法，不能先允许形成硬冲突再等待 Apply 时阻止。

规则：

1. 所有角色来自 Role Catalog。
2. 用户选中一个角色后，RoleConflictEngine 立即计算与当前选择集合不兼容的角色。
3. 硬冲突角色在 UI 中变为 disabled / unavailable，并显示原因。
4. 如果取消造成约束的角色，被禁用选项应重新变为可选。
5. UI 不得“自动取消另一个已选角色”来替用户决定；如果一个操作会使既有合法集合变非法，应拒绝该操作并解释原因。
6. 任何自由文本 role 输入框、custom label 输入框、把 GitHub label 当作角色编辑入口的设计均禁止。
7. 远端若因旧版本、人工 GitHub 修改或外部异常已经存在冲突的 `grc-role-*` labels，应进入 `InvalidRemoteRoleState`：
   - 明确展示异常；
   - 不把该冲突集合视为合法当前选择；
   - 要求用户从 Role Catalog 重新选择一个合法集合后才能 Apply 修复；
   - 不得静默任选一个角色删除。

示例：

```text
选择 Dev
→ Verify disabled: 与 Dev 冲突

选择 Repair
→ Verify disabled: 与 Repair 冲突

选择 Verify
→ Dev disabled
→ Repair disabled
```

## 7. 冲突引擎要求

冲突判断必须实现为纯函数，禁止把规则散落在 UI click handler 中。

建议接口：

```csharp
RoleValidationResult Validate(
    IReadOnlySet<MachineRole> desiredRoles,
    AssignmentState desiredState,
    MachineCapabilities capabilities);
```

返回：

- IsValid
- HardConflicts[]
- Warnings[]
- NormalizedRoles[]

UI 和 GitHub adapter 只能消费验证结果，不得自行重新解释冲突。

## 8. Apply 前检查

必须验证：

1. role enum 全部已知。
2. 没有重复。
3. 没有硬冲突。
4. Platform Test 与当前 OS/capability 一致。
5. 非 Active 状态不能产生 active runner eligibility。
6. Apply diff 只修改 `grc-role-*` labels。
7. 最新远端状态与用户开始编辑时相比若已变化，必须提示刷新/重新确认，不能静默覆盖。

## 9. 多角色示例

允许：

```text
General + CI
CI + Build
Build + Dev
CI + Verify
General + Platform Test
CI + Build + Platform Test
```

拒绝：

```text
Dev + Verify
Repair + Verify
Dev + Repair + Verify
```

## 10. 未来扩展

新增 role 必须同时提交：

- role 定义
- label
- 与全部现有 role 的兼容关系
- capability 约束
- 单元测试
- UI 文案

没有矩阵项的新 role 不得上线。
