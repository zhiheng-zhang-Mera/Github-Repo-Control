# DeepSeek 施工书 — 轻量桌面 UI / 集成验证

> 执行者按任务逐项施工；使用可用的 executing-plans 或等价流程。此文件是施工计划，不是已完成报告。

**Goal：** 用最少页面管理当前主机的仓库 Runner 与角色，直观呈现真实状态和失败。

**Architecture：** Avalonia/MVVM 消费 Core 的共享契约，不复制 GitHub、Runner 或冲突逻辑。C0 契约交付后可用明确 Demo fake 并行开发。

**Tech Stack：** .NET LTS、Avalonia、MVVM；使用 C0 锁定版本，不另起框架。

**Spec：** [architecture.md](architecture.md)、[role-model.md](role-model.md)、[shared-contracts.md](shared-contracts.md)、[acceptance.md](acceptance.md)。

## 全局约束与输入

分支建议 `feat/deepseek-desktop-phase1`，从 C0 contract SHA 开始。记录实际 host/model/base_sha/spec_sha；App 与 App.Tests 归本线，核心四工程归 Codex。Core 交付前用 fake，但不把 fake 后端重新写进 UI。

仅四个主视图：Login、Repositories、RepositoryDetail、Audit；Account Settings 为对话框。禁止项目创建/准入向导、全局调度图、批量自动启用和 Utopia 集成。英文 identifier 保留，界面中文为主。

复核重点：列表缓存不能冒充最新；角色互斥不能只靠 UI；退出不暗停/暗留登录凭据；部分失败不能变成功；本地停止与远端 offline 必须区分。对应测试分别由 D1/D2/D3 承担。

## D0 — 页面壳与契约绑定

文件：`App/Views/{LoginView,RepositoriesView,RepositoryDetailView,AuditView}.axaml`、对应 `ViewModels/`；测试 `App.Tests/NavigationTests.cs`。

输入 C0 的 IAccountService/IRepositoryService/IAssignmentService/IAuditReader/IRolePolicy；输出可启动页面与依赖注入绑定。不得直接接触 OAuth token、HTTP endpoint 或进程参数。

- [ ] 建最小导航，验证未登录只展示登录/可解释的离线状态，不混入另一个账户的 repo 数据。
- [ ] 接 C0 fake，明确 Demo 标记；生产构建无真实配置时显示 ConfigurationRequired，不自动伪造仓库。
- [ ] 运行 `dotnet test tests/GithubRepoControl.App.Tests --filter FullyQualifiedName~NavigationTests`；截图只证明 UI，不当后端验收。

## D1 — 登录与仓库列表

文件：`App/ViewModels/{LoginViewModel,RepositoriesViewModel,AccountDialogViewModel}.cs`，对应 Views；测试 `App.Tests/{LoginViewModelTests,RepositoryListTests}.cs`。

- [ ] RED：`Startup_RequestsRemoteRefresh`、`RefreshFailure_ShowsStaleTimestamp`、`PartialFirstLoad_IsNotCompleteList`、`OldAccountResult_DoesNotRender`、`Logout_WarnsRunnerContinues`。
- [ ] 使用 DeviceFlowProgress 展示 user_code、固定官方验证入口、等待/取消/拒绝/到期；不显示 token。
- [ ] 实现每次启动、新登录与手动刷新；loading 不整页闪空、无并发刷新；完整列表展示授权范围、owner/name、可见性、权限和带时间的 Runner 摘要。
- [ ] 搜索及 owner/可见性/已注册/角色过滤保持轻量；新项目发现后未托管，不自动申请 admin API 或注册。
- [ ] 运行 `dotnet test tests/GithubRepoControl.App.Tests --filter "FullyQualifiedName~LoginViewModelTests|FullyQualifiedName~RepositoryListTests"`，保留 RED/GREEN 并提交。

## D2 — 详情、角色、确认、错误和审计

文件：`App/ViewModels/{RepositoryDetailViewModel,ChangePreviewViewModel,AuditViewModel}.cs`，对应 Views/Dialogs；测试 `App.Tests/{RolePickerTests,ChangePreviewTests,FailurePresentationTests}.cs`。

详情主区只放仓库、当前主机、实际 Runner 状态、七角色、启动/待机/停用和预览。Quarantined、注销、紧急停止放“更多操作”，不建设安全仪表盘。

- [ ] RED：`AllSevenRoles_VisibleWithoutFreeText`、`DevDisablesVerify_ReverseAlsoWorks`、`UnknownRemoteRole_IsNotEditable`、`EditInvalidatesPreview`、`PartialApply_IsNeverGreenSuccess`、`RemoteOffline_LocalRunning_ShowsWarning`。
- [ ] 角色直接从 IRolePolicy 渲染，禁用项解释冲突，取消后恢复；远端非法/未知集合与合法编辑草案分区。
- [ ] Preview 展示身份、Add/Remove、停机/注册影响及安全前提；Confirm 才 Apply。字段变更/换账号/StaleState 需要重新预览。
- [ ] Busy 不自动排队；紧急停止/注销独立二次确认，明确可能中断。无网络仍保留已核实实例的本地停止入口。
- [ ] 展示 Desired 与 Observed 差异、未完成步骤及“核对状态”入口；Reconcile 只读，不偷偷重试写。用户可以取消操作，界面不宣称已撤销服务端变化。
- [ ] 公共仓库启用提示默认拒绝；外部专用环境和用户确认作为前提，显示 OperatorAttested，不给普通桌面一个万能“忽略”按钮。
- [ ] 提供从枚举与真实 OS/arch 生成的 runs-on 复制片段、官方 GitHub 设置链接；绝不提交 workflow。审计仅显示/复制脱敏结构化信息。
- [ ] 验证键盘、Tab 顺序、高 DPI、窗口缩放及长错误；运行 `dotnet test tests/GithubRepoControl.App.Tests`，提交 UI 精确 SHA。

## D3 — 一次整合与交叉验证

输入：C6 核心交付、D2 UI 交付；输出一个明确 integration SHA。使用独立 worktree 避免常驻旧构建污染；不自行改共享规范躲开错误。

- [ ] 合入两线，运行 `dotnet restore GithubRepoControl.sln --locked-mode`、`dotnet build GithubRepoControl.sln --no-restore`、`dotnet test GithubRepoControl.sln --no-build`。构建必须基于同一树，不能分别通过后相加。
- [ ] DeepSeek 复核未主导的核心：128 角色子集、服务归属/目录穿越、非 GRC 标签保护、stale/partial/crash 恢复、logout 旧 token、权限不足、busy 竞态与凭据 ACL。
- [ ] 请 Codex 按 C6 复核未主导的 UI；同机不同模型可用，记录事实，不称作跨物理主机验证。没有第二执行者就记自检并保留独立复核 NOT_RUN。
- [ ] 报告用例级结果和可复现缺陷，必要修补交给明确负责者；不按每个小提交无限往返复验，也不覆盖早期失败日志。

## D4 — 真实 E2E 与打包

文件：`evidence/phase1/desktop-e2e.md`、`evidence/phase1/acceptance-report.md`；包放发布产物目录，不把凭据/Runner 工作区提交 Git。

- [ ] 在用户授权的一台真实 Windows、两个非 Utopia 测试仓库完成启动→登录→全量刷新→选择仓库→注册→General+CI→预览→应用→双侧核对→重开恢复→改角色→审计→停止/注销。
- [ ] 用户自行准备最小受信任 workflow；实际 job 返回 runner name、OS 与测试 commit SHA，核对它在当前主机执行。不得仅用 API 标签或桌面截图证明可接单。
- [ ] 受控展示 busy 拒绝、stale/部分失败恢复、停止后不再执行、退出不暗停和重启不暗启。测试队列由用户取消，不为此增加云端回退或作业调度。
- [ ] 从 Runner 低权限身份做真实 ACL 探针；缺权限、OAuth 或仓库时记录相关 NOT_RUN/BLOCKED，不换 mock 假装通过。
- [ ] `dotnet publish src/GithubRepoControl.App -c Release -r win-x64 --self-contained true`；在无开发工具依赖的 Windows 环境启动产物，记录源码 SHA、包 SHA256、实际依赖/Runner 版本和安装步骤。
- [ ] 按 acceptance A–G 汇总；集中修补后定向重验及必要全套回归。未解决安全/数据损失问题必须 PARTIAL/BLOCKED，不为了限制复验轮次硬签 COMPLETE。

最终报告明确：已实现、单测通过、真实 GitHub/Windows 验证、未执行、失败保留、已知限制、最终 SHA。Phase 1 仅为轻量管理器，不暗示第二台实体机、Linux/macOS、项目治理或工作流安全认证完成。
