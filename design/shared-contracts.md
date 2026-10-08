# 共享契约与实现边界

**GRC-MANAGER-V1.1 · 2026-10-08**

此文档固定 Codex/Core 与 DeepSeek/UI 的最小交接面，不是插件 SDK 或远程协议。类型放在 `src/GithubRepoControl.Core/Contracts.cs`，Role Catalog 与冲突实现另放 `Roles/`。首个 C0 提交同时提供 fake 与编译测试；后续双方从这个完整 SHA 开分支。

## 一、最小类型

| 类型 | 必需内容 |
|---|---|
| AccountRef | Host（首版 github.com）、UserId、Login、SessionGeneration |
| RepositoryRef | Host、Id、Owner、Name、Visibility、Archived、CanManageRunners（未知/是/否） |
| MachineIdentity | MachineId、Hostname；身份不明不得按名称接管 |
| MachineCapabilities | ObservedOS、ObservedArchitecture；不含自动探测出的假 SDK/GPU 能力 |
| MachineRole / DesiredState | 与 role-model.md 完全一致 |
| RoleDefinition / RoleChoice | Enum、中文名、说明、固定 Label；Choice 另含 Selected、Enabled、ReasonCode |
| RoleValidationResult | IsValid、HardConflicts、Warnings；不得自动删掉冲突输入 |
| RepositorySnapshot | Account、Rows、Freshness（Fresh/Stale/Unknown）、LastSuccessfulRefreshAt、ErrorCode |
| AssignmentKey | Host、UserId、RepositoryId、MachineId |
| AssignmentSnapshot | Key、RunnerId/Name、DesiredRoles/State、ObservedLabels、RemoteStatus、Busy、LocalStatus、QuarantineLock、Freshness、ObservedAt、Revision、Problems |
| AssignmentRequest | Key、Roles、State、ExplicitUnlock（默认 false） |
| ChangePlan | PlanId、SessionGeneration、Key、ExpectedRevision、Desired、RoleDiff、RunnerActions、Warnings、CreatedAt |
| OperationResult | OperationId、Outcome、Code、Observed、CompletedSteps、RecoveryActions |
| AuditRecord | 时间、OperationId、身份、策略版本、前后实况、结果码、脱敏错误 |

RemoteStatus = `NotRegistered / Online / Offline / Unknown`；Busy = `true / false / unknown`；LocalStatus = `Absent / Stopped / Running / Unknown`。API 403/超时不能转换成 NotRegistered 或 Busy=false。Problem 可标注 InvalidRemoteRoleState、UnsupportedPolicy、OwnershipAmbiguous，不额外制造项目生命周期。

## 二、UI 仅调用这些服务

以下方法均接受 `CancellationToken ct`，返回 `Task<T>`；括号中的返回类型为 T。具体 DTO 用上一节定义，UI 不接触 token、raw HTTP 或任意命令参数。

| 服务 | 方法 |
|---|---|
| IAccountService | `SignInAsync(ct)`（AccountRef）；`GetCurrentAsync(ct)`（AccountRef?）；`SignOutAsync(ct)`（无返回值） |
| IRepositoryService | `RefreshAsync(ct)`（RepositorySnapshot）；`GetSnapshotAsync(ct)`（RepositorySnapshot，不冒充刷新） |
| IAssignmentService | `ReadAsync(AssignmentKey key, ct)`（AssignmentSnapshot）；`PreviewAsync(AssignmentRequest request, ct)`（ChangePlan）；`ApplyAsync(Guid planId, ct)`（OperationResult）；`ReconcileAsync(AssignmentKey key, ct)`（AssignmentSnapshot，只读） |
| IAssignmentService | `PreviewRemoveAsync(AssignmentKey key, ct)`（ChangePlan）；`EmergencyStopAsync(AssignmentKey key, bool userConfirmed, ct)`（OperationResult） |
| IAuditReader | `ReadRecentAsync(int limit, ct)`（IReadOnlyList<AuditRecord>，limit 1–200） |
| IRolePolicy（同步） | `Validate(roles, state, capabilities)`（RoleValidationResult）；`GetChoices(currentRoles)`（IReadOnlyList<RoleChoice>）；Catalog（IReadOnlyList<RoleDefinition>） |

Device Flow 的验证码和等待进度通过 `IAccountService.Progress` 事件返回 `DeviceFlowProgress`（UserCode、官方 VerificationUri、ExpiresAt、Stage）；不包含 device_code/token。取消登录必须终止轮询；测试 fake 默认明确标记 Demo，不混入真实账户页面。

`ApplyAsync` 仅接受本会话内、用户已确认且未被编辑/账户切换作废的 PlanId，拒绝直接从 UI 提交 raw labels。UI 确认事件不构成对恶意同身份进程的安全保证，OS 隔离仍为前提。一般角色/状态编辑和注销均走 preview；紧急停止单独二次确认。

Core 内部可使用 `IGitHubRunnerApi / ILocalRunnerManager / ICredentialStore / IOperationJournal / IAuditSink`；这些不是对项目开放的接口。它们的方法由对应施工步骤定义在所属工程，不让 App 反向依赖实现细节。

## 三、结果和后置条件

Outcome = `Succeeded / NoChange / Rejected / Partial / Failed / OutcomeUnknown`。Cancelled 是用户取消造成的错误码，不代表服务端没有发生变化。

- Succeeded/Active：远端注册及标签吻合、本机服务 Running、GitHub Online；Busy 可因刚接到工作而为 true。
- Succeeded/Standby：本机停止确认，注册仍存在，保留已批准角色；不要求远端立刻显示 Offline，但须展示观测时间。
- Succeeded/Disabled 或 Quarantined：本机停止、已知 GRC 标签清除、期望/安全锁已持久化；远端不可确认则 Partial，而不是成功。
- Succeeded/Remove：远端实例不存在且本机服务/注册清理确认；单侧成功为 Partial。
- 紧急停止：本机已停止可以报告本地停止成功，远端状态 Unknown 必须原样显示；不得把它包装成注销或标签清理成功。
- NoChange：期望和实况已经一致，零写请求、零服务重启；不重新申请注册 token。

统一 Code 至少含：`ConfigurationRequired / AuthenticationRequired / PermissionDenied / PlatformUnsupported / RoleConflict / EmptyActiveRoles / InvalidRemoteRoleState / UnsupportedPolicy / RunnerBusy / StaleState / OwnershipAmbiguous / SafetyPrerequisiteMissing / Quarantined / NetworkUnavailable / RateLimited / RegistrationUncertain / PartialApply / StopUnconfirmed / AuditUnavailable / Cancelled`。

错误必须可解释且脱敏；Code 不得只做 UI 文案，Core、fake、真实 Adapter 使用同一语义。Revision 是本地规范化快照指纹（含账户、repo、runner、标签及关键状态），不是 GitHub 原子 CAS 或分布式锁。

## 四、范围与测试约定

契约只覆盖当前主机管理。没有 `CreateProject / ApproveProject / DispatchTask / UpdateWorkflow / GrantRepositoryPermission / RemoteHostCommand` 接口，不为了未来场景预留空平台。

fake 必须能稳定触发 stale、busy、部分分页失败、部分写成功、token 失效、身份不明和 stop 超时；测试调用真实 Core，不复制冲突逻辑。生产构建默认使用真实适配器，没有凭据/环境时显示前提缺失，不自动退回假数据。
