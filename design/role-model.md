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

## 6. 冲突引擎要求

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

## 7. Apply 前检查

必须验证：

1. role enum 全部已知。
2. 没有重复。
3. 没有硬冲突。
4. Platform Test 与当前 OS/capability 一致。
5. 非 Active 状态不能产生 active runner eligibility。
6. Apply diff 只修改 `grc-role-*` labels。
7. 最新远端状态与用户开始编辑时相比若已变化，必须提示刷新/重新确认，不能静默覆盖。

## 8. 多角色示例

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

## 9. 未来扩展

新增 role 必须同时提交：

- role 定义
- label
- 与全部现有 role 的兼容关系
- capability 约束
- 单元测试
- UI 文案

没有矩阵项的新 role 不得上线。
