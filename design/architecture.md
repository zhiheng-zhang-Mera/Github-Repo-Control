# 架构设计：轻量管理器

**GRC-MANAGER-V1.1 · 2026-10-08**

## 一、职责与非目标

GRC 是单机桌面管理器：GitHub 登录、仓库发现、当前主机识别、repository-level Runner 注册/停止/启动/注销、兼容角色编辑、结果核验及本地脱敏审计。

新仓库由用户在 GitHub 正常创建；下次启动/手动刷新发现，默认未托管，经本机用户确认才注册。无需另做项目生命周期状态机。未托管不阻止 Git clone/push 或人工开发。

只支持 GitHub.com、一个当前登录账户和当前 Windows 主机。Linux/macOS 保留接口并返回 `PlatformUnsupported`；不为首版增加跨平台守护进程。Utopia 是冻结的外部参考，不修改、不依赖、不拿来试注册。

不做远程控机、全局任务/算力调度、多账户同时控制、组织级 Runner 池、权限/分支规则管理、自动改 workflow、项目创建/审批、自动云端回退、VM/容器编排、AI 审查、自进化和业务插件。

## 二、组件与可信边界

```text
Avalonia App / ViewModels（本机用户确认）
  → Core（Role Catalog + AssignmentService + 公共契约）
     ├── GitHub Adapter（只访问固定 GitHub API）
     ├── Runner Manager（仅管理明确归属的本机实例）
     └── Security（凭据存储、脱敏、本地日志）
```

保留五个工程 `App / Core / GitHub / Runner / Security`，不拆微服务。共享接口见 [shared-contracts.md](shared-contracts.md)。普通 JSON 配置和 JSONL 审计足够，不引入数据库服务、消息总线、长期运行的控制器后台或 localhost 管理 API。

控制器不 clone、不加载或执行被管理仓库的源码、脚本、配置来决定权限；官方 Runner 在其独立执行身份下执行已授权工作流，这是不同的执行边界。GRC 不从项目文件读取角色策略。角色规范编译进 Core，不接受工作流反向修改。

## 三、登录、权限与账户切换

首版保留 OAuth App Device Flow：发行者提供已启用 Device Flow 的公开 client id，桌面不带 client secret。没有 client id 是 `ConfigurationRequired`，可做 fake 开发，不能伪造登录。Device Flow 的 interval、slow_down、拒绝和到期均按实际响应处理。[S1]

repository-level Runner 管理要求对应仓库管理权限；OAuth `repo` scope 覆盖面较广，UI 必须明示它不只是“读取列表”。不额外请求 `admin:org`、`workflow` 或 `delete_repo`，不因拥有广 scope 就允许控制器写代码。列出仓库不代表拥有 Runner 管理权限；权限不足只读，不循环重试管理 API。[S2]

token 进入当前用户的 Windows 安全凭据存储。刷新令牌仅在所选 OAuth 配置和官方返回确实支持时使用，不把 GitHub App 的能力当成所有 OAuth App 的保证；不存在可用 refresh token 时重新登录。每次获得新 token 重新读取用户 numeric id，避免串号。

本地状态以 GitHub host + user id 分区。logout 取消请求、使所有 preview 失效、清除安全存储的登录凭据与内存引用，禁止后续调用继续使用旧 token；已送达服务端的请求不能被承诺撤销，须在下次登录后核对。不得宣称 .NET 托管内存已可证明逐字节擦除。

logout 不等于远端撤销 OAuth 授权，也不等于停止 Runner。对话框明确说明仍在运行的数量，提供独立停止操作；需要撤销授权时打开 GitHub 官方授权设置，首版不新增撤销授权 API。

## 四、仓库列表与身份

启动恢复有效登录态、新登录成功、手动刷新均调用同一全量分页流程；只有全部页面成功后才原子替换快照并更新 `lastSuccessfulRefreshAt`。分页中断保留旧完整快照且标 `Stale`，初次失败显示 `Unknown`，不能把缺页仓库当作删除。新仓库不自动注册；消失、改名或权限变化只更新展示并失效旧 preview，不自动删除本机目录。

仓库以 GitHub numeric repo id 作为身份，owner/name 是可变显示名与当前 API 路由。转移、改名后按 id 对齐，变更前重新确认当前路径和权限。账号切换清除页面快照，不能接管另一账户的 Runner。

Runner 详情仅在打开详情、用户刷新或操作后读取；列表可展示带时间的缓存摘要。只对已确认具备权限的仓库调用 Runner API；未知不是未注册，offline 也不是 stopped。列表保留搜索、owner、可见性、已注册和角色过滤，不做大型监控面板。

主机首次生成并持久化 `machine_id` UUID；hostname、OS、arch 单独刷新。UUID 不因重命名改变，也不作为安全身份证明。重装/复制配置造成身份不明时不可按 hostname 猜测或接管旧实例。

## 五、Runner 实例与最低隔离

本地 registry key 为 `(github_host, user_id, repo_id, machine_id)`；保存 runner id/name、服务名、受控实例路径。实例目录使用 ID 而非 owner__repo 拼接：

```text
<controller-data>/accounts/<user-id>/registry.json   # 控制器专属 ACL
<runner-root>/<user-id>/<repo-id>/                  # 每仓库独立配置/工作目录
```

两者均在项目工作区之外；路径规范化后必须仍位于受控根，拒绝目录穿越、符号链接和 Windows reparse point 越界。远端 Runner 名仅帮助定位，归属还需本地 registry、repo id、machine id 与服务配置一致；未知实例只展示，不自动收养、删除或使用 `--replace` 覆盖。

Windows 使用官方支持的 Runner 服务模式，服务启动类型为手动。Runner 必须以不同于 GRC 交互用户的低权限身份运行，不得以该用户、LocalSystem 或管理员身份执行业务作业。管理员可预置执行账户及 ACL；首版只引导/检查这些前提，不开发账户管理系统。GRC UI 常态非管理员，安装/服务操作需要权限时使用受限的一次性提升流程，仅允许固定 GRC 服务及受控路径，不暴露任意命令执行；提升端不得获取控制器 OAuth token。[S4]

必须以 Runner 执行身份实测：无法读取控制器安全存储/registry/日志，无法修改已安装的控制器程序、策略或 registry。目录分开不是沙箱；多个 Runner 共用执行身份时也不构成跨仓库安全隔离。攻击者已有本机管理员权限的情形不在保证范围内。[S3]

Runner 仅从 GitHub 官方发布渠道准备，记录版本/平台/来源和可信校验摘要，校验失败不执行；不可下载项目提供的安装脚本。注册/移除 token 为短期秘密，不落日志/配置/命令历史；仅在受控安装进程必需范围传递，禁止通用进程命令行日志。官方 Runner 自身持久化的运行凭据由其实例 ACL 保护，不能把“注册 token 不落盘”误写成“Runner 没有持久凭据”。控制器 OAuth token 永不传入 Runner 进程或环境。[S2]

## 六、执行安全范围

首版只接收用户明确认可的受信任工作流，不执行不可信 PR 作为安全测试。所有仓库的内容对控制器仍是不可信输入。[S3]

私人仓库启用前必须通过执行身份/ACL 检查并说明其持久执行风险。公共仓库默认 `SafetyPrerequisiteMissing`；需用户已在外部准备专用执行主机或隔离 VM、明确确认只运行受信任作业，且满足同样的身份隔离后才能显式启用。普通日常桌面不能仅点“忽略警告”绕过。环境用途声明记 `OperatorAttested`，GRC 不声称自动证明 VM、网络隔离或工作流可信；无法满足则继续仅查看/编辑草案/停止/清理。

GRC 不创建隔离环境、不部署 Kubernetes、不检测所有 workflow 风险、不承诺防止 GitHub 管理员绕过。安全承诺限于 GRC 自身入口和已验证的本机权限边界。紧急停止/减少权限不因新增启用门槛而被阻塞。

## 七、角色、路由和实际状态

七个角色及矩阵见 [role-model.md](role-model.md)。只有已知目录项可编辑；标签名称比较大小写不敏感，写入统一小写。只增删本版本已知 `grc-role-*` 标签，保留所有其他 custom/read-only 标签；未知 GRC 标签视为 `UnsupportedPolicy`，不擅自删除。[S2]

详情只生成来自 Role Catalog 和本机真实 OS/arch 的可复制 `runs-on` 片段，不提交到项目：

```yaml
runs-on: [self-hosted, windows, x64, grc-role-ci]
```

它仅说明路由条件；工作流必须实际使用标签。默认 `self-hosted`/OS 标签可能匹配不带 GRC 角色的工作流，所以删标签不是安全停用。`verify` 仅为声明的用途，不证明独立审查；普通 `git push` 也不由这些标签决定。[S5]

`DesiredState`、GitHub `RemoteStatus/Busy`、本机 `LocalStatus`、角色和读取新鲜度分别记录。仅本机服务停止确认可证明本实例停止；远端 offline 可能只是断网。仅远端标签更新成功不得显示整个操作成功。详细后置条件见共享契约。

## 八、计划、确认、执行、核验

首版单实例运行，并串行执行本机写操作；没有分布式锁服务。所有操作都有不含秘密的 `operation_id` 与先落盘的操作记录。写日志失败时拒绝新授权/启动；紧急停止仍尽力执行并显著报告日志失败。

1. 读取当前账户、仓库权限、可见性、归属、本机服务及 GitHub Runner 状态；纯函数验证角色。未知/过期状态不能据以新增授权。
2. 生成 preview：绑定账户/session generation、repo id、machine id、runner id、策略版本、完整标签集合和目标变更。展示需要停止/注册/启动的影响；无差异返回 NoChange，不重启、不写 API。
3. 用户确认时再读实况；账户、仓库、标签、权限或本地归属已变则 `StaleState`，要求重新预览，不自动覆盖。预览后字段改变同样失效。
4. 已观察到 busy 的普通角色变更/停止/注销直接 `RunnerBusy`，不排队等待。空闲时先停止并确认本地实例，再执行标签和配置变更；移除旧冲突角色后确认，再添加新角色，不能中间形成 Dev+Verify。
5. 若目标 Active，复核角色、权限、隔离前提及远端标签后启动；回读本机与 GitHub 状态，全部满足才 `Succeeded`。其他目标确认停止并落实对应标签/锁状态。
6. 请求超时或操作部分成功标 `Partial/OutcomeUnknown`，保留已发生步骤；不自动回滚到可能过时的授权、不自动重启、不盲重试注册。用户点击“核对状态”只读取，恢复变更须另行预览和确认。

GitHub 标签 API 不提供本流程跨多个请求的原子事务保证；本地锁+前后读取只是冲突检测，不能阻止 GitHub 网页/API 同时改动。空闲检测与停止之间也有接单竞态，首版不声称零中断或可靠排空；检测到竞态则停止后续变更，显示任务可能受影响并保留证据。[S2]

## 九、生命周期与恢复

| 动作/情况 | 第一阶段确定行为 |
|---|---|
| 首次注册 | 先记录计划与唯一归属，再注册官方实例；确认远端 id 后同步标签；批准 Active 且核验通过才启动 |
| 注册超时/崩溃 | 标 OutcomeUnknown；按原操作与精确归属查找部分实例，先核对，不重复注册/覆盖 |
| Standby | 停止本地服务，保留注册与已批准角色；标签可保留，绝不把标签存在等同接单 |
| Disabled | 停止服务，移除本版已知 GRC 角色标签；保留注册、用户期望和诊断信息 |
| Quarantined | Disabled 效果加持久本地安全锁；只有用户明确解除后才可重新预览启用，不自动检测威胁 |
| 紧急停止 | 独立二次确认，允许 busy/离线/未登录；只操作归属已核实的本机服务，告知可能中断任务，不虚报远端清理完成 |
| 注销 | 确认 idle → 停止 → 官方注销/远端移除 → 双侧核对；远端已删但本地残留单独报告，禁止直接删未知目录 |
| 关闭窗口/退出登录 | 不改变已运行 Runner；告知仍运行。未完成操作落盘后结束控制器，不假装取消了已送达请求 |
| 系统重启 | 服务手动启动；GRC 启动仅读状态，观察到旧 Desired Active + Local Stopped 显示未运行，不自动恢复 |
| 断网/权限消失 | 禁止新增授权；既有服务运行态如实显示，允许本地停止；不删数据/不改仓库权限 |
| 本地日志/配置损坏 | 只读恢复，禁止启动或自动接管；保留原文件，不覆盖为“空配置” |

检查服务状态/网络回读采用有界等待：服务控制最多 30 秒，普通 API 单次超时 30 秒；GET 仅在临时故障下最多重试两次，并遵守更长的 Retry-After；需要等待更久就显示可重试状态，不冻结界面。非幂等写操作不自动重发。Device Flow 使用其自身到期和 interval，不套用普通 GET 重试。

## 十、审计与故障信息

使用有界轮转 JSONL + 小型未完成操作记录，不建设论文素材系统。单日志 5 MiB、最多 5 个轮转文件；未完成操作记录独立保留直到核对，不因轮转丢掉恢复线索。

记录 UTC 时间、operation id、user/repo/machine/runner id、策略版本、before/desired/observed、阶段、结果码、脱敏错误、源码/应用版本。轮转只清理自身已关闭日志，不扫描/删除项目目录。

不记录 Authorization、token、device_code、refresh token、注册/移除 token、完整进程参数或原始 HTTP body。错误详情也先结构化白名单再脱敏。日志是本机可审阅记录，不宣称防管理员篡改、全历史永久保存或第三方独立证据。

## 十一、验收范围

必须有真实 Windows Desktop → GitHub 登录 → 两个授权测试仓库的独立实例 → 标签/服务核验 → 实际受信任 workflow 作业，不能以截图、API mock 或标签已写入冒充能接单。负向安全场景使用自有受控 fixture，不执行真实恶意外部代码。详细用例见 [acceptance.md](acceptance.md)。

两台物理机验证是可选，不是这个轻量工具的前置条件；至少一台真实 Windows 的集成验证与未主导代码的交叉复核必需。Linux/macOS、自动调度和项目内任务独立性不在 Phase 1 COMPLETE 声明范围。

## 十二、官方依据

核对日期：2026-10-08。以下仅支撑平台事实；超时、日志上限、公共仓库启用门槛等是本项目的设计选择，不是 GitHub 保证。施工时记录真实 API/依赖/Runner 版本。

- [S1 — OAuth App 授权与 Device Flow](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
- [S2 — repository-level Runner、token 与标签 API](https://docs.github.com/en/rest/actions/self-hosted-runners)
- [S3 — Self-hosted Runner 安全边界](https://docs.github.com/en/actions/reference/security/secure-use)
- [S4 — 官方 Runner 服务配置](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/configure-the-application)
- [S5 — runs-on 与标签路由](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job)
