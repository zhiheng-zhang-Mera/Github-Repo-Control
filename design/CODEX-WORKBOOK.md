# Codex 施工书 — Core / GitHub / Runner

## 任务定位

Codex 负责第一阶段的核心域、GitHub 集成、runner 生命周期和安全基础设施。

不得修改共享设计来降低施工难度。发现设计无法实现时，记录 blocker 与建议，不得静默改变语义。

## 分支

建议：

```text
feat/codex-core-phase1
```

不得直接在 main 上边施工边改共享规范。

## 输入

开工前完整阅读：

- `README.md`
- `design/README.md`
- `design/architecture.md`
- `design/role-model.md`
- `design/acceptance.md`

## C0 — 初始化桌面项目

建立：

```text
src/
  GithubRepoControl.App/
  GithubRepoControl.Core/
  GithubRepoControl.GitHub/
  GithubRepoControl.Runner/
  GithubRepoControl.Security/

tests/
  GithubRepoControl.Core.Tests/
  GithubRepoControl.GitHub.Tests/
  GithubRepoControl.Runner.Tests/
```

要求：

- Avalonia + MVVM。
- UI 不直接调用 raw HTTP。
- Core 不依赖 Avalonia。
- GitHub adapter 可 mock。
- runner process 操作封装为接口。

验收：solution 可 restore/build/test。

## C1 — Core domain

实现至少：

- MachineIdentity
- MachineCapabilities
- RepositoryRef
- MachineRole
- RoleCatalog
- AssignmentState
- RunnerAssignment
- RoleValidationResult
- RoleConflictEngine

角色冲突必须完整编码 `role-model.md` 矩阵。

`RoleCatalog` 必须是受控、枚举式的数据源，向 UI 暴露：

- role id
- display name/key
- description
- GitHub label
- capability constraints
- compatibility information / validation hook

禁止提供“解析任意字符串为新角色”的隐式扩展路径。未知 role 必须被拒绝。

写参数化单元测试覆盖所有 pair，并增加：

- 当前合法选择集合 → 可选/不可选角色集合。
- 未知角色拒绝。
- API 绕过 UI 时冲突组合仍拒绝。
- 远端冲突 labels → InvalidRemoteRoleState。

验收：Dev+Verify、Repair+Verify 必须失败；General+CI、CI+Build 必须通过。

## C2 — Machine Identity

实现稳定 machine_id：

- 第一次启动生成 UUID。
- 存本应用 data directory。
- hostname/OS/arch 独立刷新。
- machine_id 不因 hostname 改动而变化。

不要使用 MAC 地址作为唯一身份。

验收：模拟 hostname 变化仍保持 machine_id。

## C3 — Secure Credential Store

定义：

```text
ICredentialStore
```

Windows 实现必须使用 OS 安全凭据能力，不得明文文件保存 token。

Linux/macOS 暂可提供接口与明确 NotSupported/后续实现，但不得退化成明文。

实现日志 redaction utility。

验收：仓库全文搜索不能出现测试之外的真实 token 格式；日志测试确保 Authorization 值被清除。

## C4 — GitHub 登录

实现 GitHub Device Flow。

要求：

- client id 来自受控应用配置，不包含 client secret。
- 显示 user_code 与验证 URI。
- 遵守 GitHub 返回的 polling interval。
- 处理 authorization_pending / slow_down / expired / denied。
- 成功后读当前用户信息。
- token 进入 ICredentialStore。
- 支持 logout。

不要把 device_code/access_token 打日志。

验收：mock flow + 至少一次真实登录证据。

## C5 — Repository Discovery

实现分页读取已授权仓库，并把远端刷新作为明确生命周期，而不是一次性初始化数据。

必须实现：

- `RefreshRepositoriesAsync`（或等价 service contract）。
- 应用启动恢复有效 session 后自动调用一次。
- 新登录成功后自动调用一次。
- UI 手动刷新调用同一 service。
- 刷新请求支持 cancellation，禁止并发重复刷新造成竞态。
- 保存 `lastSuccessfulRefreshAt`。
- 失败时返回可区分的 stale-cache 状态，不得把缓存伪装成 fresh。
- 成功结果需要处理新增 repo、消失 repo、权限变化。

实现分页读取已授权仓库：

字段至少：

- id
- owner
- name
- full_name
- visibility/private
- archived
- permissions

支持取消、刷新、错误处理。

不要 clone。

验收：真实账户 repo 列表与 GitHub 可见范围一致，并明确“授权范围”。

## C6 — GitHub Runner API Adapter

实现 repository runner 所需 API 抽象：

- list runners
- get/list labels
- create registration token
- set/add/remove controller-owned labels
- remove runner
- refresh runner state

必须保留其它非 `grc-role-*` custom labels。

对 401/403/404/409/422 做结构化错误。

验收：mock contract tests。

## C7 — Local Runner Instance Manager

实现每 repo 独立目录：

```text
runners/<owner>__<repo>/
```

需要：

- 安装/准备 runner binary 的受控流程。
- repository-level registration。
- start/stop/status。
- remove 清理。
- process/service abstraction。

安全要求：

- registration token 只进子进程参数/标准输入所需最小范围。
- 禁止写 token 到 stdout log。
- repo 名必须 path sanitize，禁止 path traversal。

验收：两个测试 repo 的实例目录互相独立。

## C8 — Assignment Service

Assignment Service 只接受 `MachineRole` / Role Catalog 中的已知枚举值，不接受 raw string 角色或 raw label 作为 desired role 输入。

实现用户 Apply 所需业务事务：

```text
Load remote
→ Validate desired
→ Calculate diff
→ Re-check remote
→ Apply
→ Verify remote
→ Audit
```

若编辑期间远端发生变化，返回 StaleState，要求 UI 刷新，不静默覆盖。

若 repo 尚未注册且 desired state 为 Active + 至少一个 role，则走注册。

## C9 — Audit

实现本地 append-only 风格审计记录：

- before
- after
- repo
- machine
- action
- success/failure
- sanitized error

不得包含 secret。

## C10 — 交付给 DeepSeek

提供稳定接口与 README：

- ViewModel 所需 services
- DTO
- error types
- role validation API
- fake/mock provider

不要要求 UI 层知道 runner token 或 GitHub raw endpoint。

## Codex 终验

提交施工报告：

```text
Implemented:
Tests:
Real GitHub evidence:
Known limitations:
Security checks:
Final commit SHA:
```

必须逐项对照 `acceptance.md`，未完成项明确标红，不得用“基本完成”代替。
