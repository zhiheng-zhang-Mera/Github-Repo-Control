# DeepSeek 施工书 — Desktop UI / UX / Independent Verification

## 任务定位

DeepSeek 负责桌面 UI、交互状态机、角色编辑体验，并对 Codex 核心实现做独立验证。

原则：

- 不复制一套 GitHub/runner 核心逻辑到 UI。
- 不通过 UI 绕过 RoleConflictEngine。
- 不修改安全规则来让界面“更顺”。
- Codex 核心尚未就绪时使用 fake service 开发，不直接重写 backend。

## 分支

建议：

```text
feat/deepseek-desktop-phase1
```

最终通过明确 integration commit/PR 合并。

## 输入

完整阅读：

- `README.md`
- `design/README.md`
- `design/architecture.md`
- `design/role-model.md`
- `design/acceptance.md`
- Codex 暴露的 service contract

## D0 — UI 信息架构

至少设计以下页面：

```text
Login
  ↓
Repositories
  ↓
Repository Detail
  ├── Current Host
  ├── Runner State
  ├── Roles
  └── Apply Preview

Audit
Settings / Account
```

第一阶段禁止加入与主链无关的大型 dashboard。

## D1 — Login 页面

包含：

- 应用定位。
- “登录 GitHub”按钮。
- Device Flow user code。
- 打开浏览器/复制验证码。
- waiting / success / denied / expired 状态。
- 当前账户。
- logout。

不得显示 access token。

## D2 — Repository List

显示：

- owner/name
- public/private
- archived
- 当前主机 runner 状态
- 当前角色摘要
- 权限不足标识

支持：

- search
- owner filter
- visibility filter
- registered filter
- refresh

大量 repo 时不得一次性阻塞 UI thread。

## D3 — Repository Detail

核心布局建议：

```text
Repository
Owner / Visibility / Permission

Current Host
hostname / OS / arch / machine_id-short

Runner
Registered / Online / Offline / Busy

Roles
[ ] General
[ ] CI
[ ] Build
[ ] Dev
[ ] Verify
[ ] Repair
[ ] Platform Test

State
Active / Standby / Disabled / Quarantined

[Preview Changes]
```

角色必须支持多选。

## D4 — 冲突交互

用户选择冲突组合时立即显示本地冲突，但不要自行写另一套规则。

例如：

```text
☑ Dev
☑ Verify

无法应用：
Verify 要求独立验证，不能与同仓库 Dev 同时存在。
```

按钮进入 disabled 状态。

同时提供“取消 Verify”或“取消 Dev”的纯 UI 快捷操作；不能自动替用户决定。

## D5 — Apply Preview

点击 Apply 前必须展示 diff：

```text
Utopia / Alien

Add:
+ grc-role-ci
+ grc-role-build

Remove:
- grc-role-general

Runner:
Registered → Registered

[Cancel] [Confirm]
```

如果涉及首次注册或 remove runner，必须额外说明影响。

确认后才调用 AssignmentService。

## D6 — Busy / stale / failure

必须处理：

- API loading
- runner busy
- remote changed
- permission denied
- offline
- registration failed
- GitHub rate limit
- network disconnected

StaleState 必须要求刷新后重新确认。

失败后 UI 不得保留“已成功”的假状态。

## D7 — Audit 视图

至少可查看：

- time
- repo
- action
- role change
- result

错误详情必须脱敏。

提供 copy sanitized diagnostic，不包含 token。

## D8 — Accessibility / Desktop quality

- 键盘可操作。
- 合理 tab order。
- loading 不冻结窗口。
- 窗口缩放布局不破裂。
- 高 DPI 可读。
- destructive action 有二次确认。
- role label 使用人类可读中文/英文名称，不直接把内部 label 当主要 UI。

根 README 保持中文；应用首版 UI 可中文为主，内部 identifier 用英文。

## D9 — 独立验证 Codex

DeepSeek 不仅做 UI，还必须针对核心做 adversarial review：

1. repo 名含特殊字符是否可 path traversal。
2. token 是否可能进入 log。
3. Apply 是否会删除非 GRC labels。
4. stale remote state 是否被覆盖。
5. Verify/Dev 冲突是否能通过 API 旁路。
6. 两 repo runner instance 是否隔离。
7. 未授权 repo 是否错误调用 admin API。
8. logout 后旧 token 是否仍可被 service 使用。

发现问题提交明确 bug / fix commit，不要仅写评论。

## D10 — E2E

至少完成真实桌面主链：

```text
Launch
→ Login
→ Repositories loaded
→ Select repo
→ Register current host
→ Select General + CI
→ Preview
→ Apply
→ GitHub confirms labels
→ Restart app
→ State restored
→ Change roles
→ Audit visible
```

再完成冲突案例：

```text
Dev + Verify
→ blocked before GitHub write
```

## DeepSeek 终验报告

输出：

```text
UI implemented:
E2E evidence:
Independent bugs found:
Bugs fixed:
Remaining blockers:
Acceptance checklist:
Final commit SHA:
```

不得只以截图证明后端成功；runner/label 状态必须有 API 或 GitHub 侧证据。
