# Codex 施工书 — 轻量管理器 Core / GitHub / Runner

> 执行者按任务逐项施工；使用可用的 executing-plans 或等价流程。此文件是施工计划，不是已完成报告。

**Goal：** 实现可真实管理当前 Windows 主机 Runner 的最小核心，不构建项目治理平台。

**Architecture：** Core 管角色与操作流程，GitHub/Runner/Security 分别承担适配。App 只依赖共享接口；C0 先交契约和 fake，后续 UI 与核心并行。

**Tech Stack：** .NET LTS、Avalonia、MVVM、GitHub REST、Windows 安全存储与官方 Runner 服务。施工时锁定实际稳定版本。

**Spec：** [architecture.md](architecture.md)、[role-model.md](role-model.md)、[shared-contracts.md](shared-contracts.md)、[acceptance.md](acceptance.md)。

## 全局约束与交接

仅当前主机、单账户、repository-level Runner；全部七角色列表选择且双层拒绝冲突；启动/手动全量刷新；不改 Utopia、不执行项目配置、不自动改 workflow、不做远程调度。所有失败如实记录，不更改规范逃避验收。

分支建议 `feat/codex-core-phase1`，记录实际 base_sha/spec_sha/host/model。C0 后 App 归 DeepSeek，核心四工程归 Codex；共享类型更改先同步到双方。不要在 main 边开发边修改规范。

复核关注：部分分页不删仓库；换账号不串 token/plan；busy/stop 竞态不假排空；部分写成功不假原子回滚；Runner 身份无法访问控制器凭据。这五项分别落入 C2/C3/C4 的测试。

每个实现任务执行“写失败用例 → 记录失败 → 最小实现 → 重跑通过 → 提交”；纯项目初始化不伪造失败测试。以下测试名为具体交付要求，最终报告保存实际命令与结果。

## C0 — 工程骨架、公共契约与 fake（先交付）

创建 `GithubRepoControl.sln`、`global.json`、`src/GithubRepoControl.{App,Core,GitHub,Runner,Security}/`、`tests/GithubRepoControl.{Core,GitHub,Runner,App}.Tests/`。目标框架由锁定 SDK 支持的 LTS 决定；提交 NuGet 锁文件，不依赖开发机全局状态。

创建 `Core/Contracts.cs` 与 `App.Tests/Fakes/ManagerFake.cs`，准确实现 shared-contracts 的所有 UI 接口；内部 GitHub/Runner/Security 适配器保持接口边界。App 默认真实模式，显式 Demo 才允许 fake。

- [ ] 建立最小 solution，运行 `dotnet restore GithubRepoControl.sln`、`dotnet build GithubRepoControl.sln --no-restore`、`dotnet test GithubRepoControl.sln --no-build`，记录可编译/测试发现结果。
- [ ] 在 `Core.Tests/ContractShapeTests.cs` 验证三态 Busy、Freshness、PlanId 绑定和所有枚举；fake 可触发 busy/stale/分页失败/部分写/登录失效。
- [ ] 交付完整 contract SHA 与依赖锁定说明。DeepSeek 从此提交开始，不必等 C5；后端缺少时提供明确错误，不能在生产自动切 fake。

## C1 — 角色与身份

文件：`Core/Roles/{RoleCatalog,RoleConflictEngine}.cs`、`Core/MachineIdentity.cs`；测试 `Core.Tests/{RoleMatrixTests,MachineIdentityTests}.cs`。

输出：IRolePolicy 的 Validate/GetChoices/Catalog；持久 MachineIdentity 和 AssignmentKey。输入只接受已知 MachineRole、DesiredState 和实际 MachineCapabilities，不接受 raw label。

- [ ] 先写 `All128Subsets_80Compatible48Conflicting`、`ActiveEmpty_IsRejected`、`DisabledConflict_IsRejected`、`UnknownEnum_IsRejected`、`HostnameRename_PreservesMachineId`。
- [ ] 运行 `dotnet test tests/GithubRepoControl.Core.Tests --filter "FullyQualifiedName~RoleMatrixTests|FullyQualifiedName~MachineIdentityTests"`，保存 RED，再最小实现后保存 GREEN。
- [ ] 覆盖 GetChoices 的双向禁用/恢复、远端已知冲突与未知 GRC label；提交，不新增角色或 SDK/GPU 探测。

## C2 — 登录、账户隔离与列表

文件：`GitHub/{DeviceFlowAccountService,RepositoryService,GitHubApiClient}.cs`、`Security/{WindowsCredentialStore,SecretRedactor}.cs`；测试 `GitHub.Tests/{DeviceFlowTests,RepositoryRefreshTests,AccountIsolationTests}.cs`。

输出 IAccountService/IRepositoryService，进度事件不含秘密；输入配置只有公开 client id 与固定 GitHub host。adapter 明确权限错误，禁止 scope 膨胀。

- [ ] 写 RED：`MiddlePageFails_KeepsStaleSnapshot`、`NewLogin_RefreshesAllPages`、`OldSessionCallback_IsDiscarded`、`Logout_InvalidatesPlansAndCredentials`、`SlowDown_RespectsInterval`、`TokenlessResponse_DoesNotInventRefresh`。
- [ ] 实现完整分页快照替换、真实 numeric 用户/仓库身份、按需 Runner 详情权限预检和有界重试；仅在实际支持时使用 refresh token，否则重新登录。
- [ ] 运行 `dotnet test tests/GithubRepoControl.GitHub.Tests`；真实登录需要授权时记录 BLOCKED，不妨碍其它任务。提交 RED/GREEN 和脱敏日志。

## C3 — 官方 Runner 与本机服务

文件：`Runner/{LocalRunnerManager,RunnerRegistry,RunnerPackageVerifier,WindowsServiceController}.cs`；测试 `Runner.Tests/{InstanceIsolationTests,ServiceLifecycleTests,OwnershipTests}.cs`。

内部 ILocalRunnerManager 明确提供 `InspectAsync(key, ct)`、`PrepareAsync(key, ct)`、`RegisterAsync(key, shortLivedRegistrationToken, ct)`、`StartAsync(key, ct)`、`StopAsync(key, emergency, ct)`、`RemoveAsync(key, shortLivedRemoveToken, ct)`；返回不含秘密的本地实况/结果，令牌从不跨到 App。Prepare 只接受可信官方发布信息，不接受项目脚本。

- [ ] RED：`TwoRepoIds_HaveSeparatePaths`、`RenamedRepo_DoesNotCreateSecondInstance`、`UnknownOwnership_DeniesMutation`、`ReparseEscape_IsRejected`、`Busy_NormalStopRejected`、`IdleToBusyRace_IsReported`、`StopTimeout_DoesNotClaimStopped`。
- [ ] 实现受控实例、官方二进制校验、独立低权限服务身份、手动启动类型及一次性受限提升。身份/ACL 未满足拒绝 Active，不做自动账户/VM 管理。
- [ ] 用 Runner 实际执行身份做控制器文件/凭据拒绝访问探针；未具备环境只能标 NOT_RUN。OAuth token 不进入 Runner 环境。
- [ ] 运行 `dotnet test tests/GithubRepoControl.Runner.Tests`，记录 Windows 服务测试与 mock 的区别，提交。

## C4 — Assignment、标签与操作日志

文件：`Core/{AssignmentService,ChangePlanner}.cs`、`GitHub/GitHubRunnerApi.cs`、`Security/{OperationJournal,AuditLog}.cs`；测试 `Core.Tests/AssignmentTests.cs`、`GitHub.Tests/RunnerLabelTests.cs`。

输出 IAssignmentService 全部方法；内部 GitHub adapter 封装 List/GetRunner、ListLabels、AddKnownLabels、RemoveKnownLabel、CreateRegistrationToken、CreateRemoveToken、DeleteRunner。禁止 replace-all/clear-all labels；404 必须回读区分原因，不统一当成功。

- [ ] RED：`ExternalLabelChange_RejectsStalePlan`、`NoChange_PerformsZeroWrites`、`ConflictRemovalFails_DoesNotAddVerify`、`PartialWrite_LeavesStoppedAndReportsObserved`、`RegistrationTimeout_DoesNotRetryCreate`、`AuditUnavailable_BlocksGrantButNotEmergencyStop`。
- [ ] 实现 preview/session/revision 绑定、串行写、先日志后授权、停止后移除冲突标签再新增、后置条件核验、只读 Reconcile；失败不自动回滚/重启。
- [ ] 增加崩溃后核对、取消发生在远端写之后、未知 GRC 标签、网络失联及公共仓库启用门槛测试。
- [ ] 运行 `dotnet test GithubRepoControl.sln`；检查 token 不进审计、诊断、HTTP 或进程参数日志；提交。

## C5 — 组合核心回归

文件：`Runner.Tests/ManagerFlowTests.cs`、`Core.Tests/RecoveryTests.cs`，只补联动缺陷，不增加功能。

- [ ] 通过真实 Core + 可控 adapter 驱动 Register → Active → Standby → Disabled → Quarantined → 用户解锁 → Remove；验证共享契约每一项后置条件。
- [ ] 验证 App 关闭/logout 不暗停、重启不暗启、离线仍能本地紧急停止，以及恢复不能误删另一个实例。
- [ ] 固定精确核心 SHA，提交 `evidence/phase1/core-report.md`，逐项映射 acceptance A–G。只写已运行结果，日志脱敏并有界保存。

## C6 — 集成交付与交叉复核

- [ ] 向 DeepSeek 交付 C0 contract SHA、最终 core SHA、实际 API/SDK/Runner 版本、测试命令、阻塞和可复现失败。
- [ ] 消费 D3 的明确 integration SHA；复核自己未主导的 UI：无假成功、无自由文本角色入口、确认/退出/错误可操作。
- [ ] 配合 D4 真实 Windows 与授权测试仓库验证；不要自行改项目 workflow、注册 Utopia、创建仓库或扩大 GitHub 权限。
- [ ] 最后集中修补与定向回归，保留之前失败。新的 scope 缺口留作建议；安全/数据损失缺陷未解决只能 PARTIAL/BLOCKED。

报告至少包括 `spec_sha / base_sha / final_sha / host / model / implemented / tests / real_evidence / failures / remaining / artifact_hashes`。不能把文档完成、单元测试通过或标签写入成功称为 Phase 1 COMPLETE。
