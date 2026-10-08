# 第一阶段验收标准

**GRC-MANAGER-V1.1 · 2026-10-08**

本清单未勾选代表规范要求，不代表测试结果。每项在交付报告中引用源码 SHA、命令/操作步骤、结果和脱敏证据；`PASS / FAIL / NOT_RUN` 分开。单元测试可用 mock；主链真实端到端不能以 mock 替代。

## A. 范围与桌面入口（C0 / D0）

- [ ] A01 Windows 桌面可启动，只有登录、仓库列表、详情、审计四个主视图及轻量账户设置；中文主页保留。
- [ ] A02 新仓库在刷新后出现但不自动注册/启动；手动完成首次配置即可接入，无项目创建/审批/准入平台。
- [ ] A03 不修改 Utopia，不需要它运行；无项目脚本控制角色、自动改 workflow、远程控机和普通 Git push 拦截。
- [ ] A04 Linux/macOS 未实现能力明确 NotSupported/PlatformUnsupported，不回退到明文凭据或假成功。

## B. 账户与仓库（C2 / D1）

- [ ] B01 真实 Device Flow；显示正确 GitHub 用户。缺 client id、拒绝、到期、slow_down、取消均可解释，无 secret 打包。
- [ ] B02 登录/启动/手动刷新都实际分页访问 GitHub；重复刷新不并发提交旧结果，UI 不冻结。
- [ ] B03 中间页失败时旧完整快照为 Stale，初次失败为 Unknown；不能丢仓库或更新最后成功时间。
- [ ] B04 搜索及 owner/可见性/注册/角色过滤可用；Runner 详情按需读取，权限不足不循环打管理 API。
- [ ] B05 仓库按 numeric id 对齐，改名/转移/权限变化使旧 preview 失效；未列出不自动删除 Runner。
- [ ] B06 换账号/退出后旧 plan、请求回调和 token 不再被用于后续调用；缓存不串号。
- [ ] B07 logout 提示 Runner 可能继续运行；清除本地登录凭据但不谎称已撤销远端授权或停止 Runner。

## C. 角色目录（C1 / D2）

- [ ] C01 七角色全部可见，无自由文本角色、custom label 或隐藏扩展入口；项目配置不能新增角色。
- [ ] C02 Dev/Repair 与 Verify 双向立即禁用，取消后恢复；不自动取消其他选项。
- [ ] C03 枚举 128 子集：80 兼容、48 角色冲突；单独覆盖未知 enum、Active 空集合以及非 Active 下仍拒绝冲突。
- [ ] C04 ViewModel/AssignmentService 绕过 UI 的冲突在任何远端写前拒绝。
- [ ] C05 已知非法远端集合分区展示并显式修复；未知 GRC 标签返回 UnsupportedPolicy，不误删。
- [ ] C06 普通 custom/read-only 标签保持不变，大小写归一不重复创建；移除冲突旧标签后才能加新标签。
- [ ] C07 UI 明示 label≠权限、Verify≠独立性、General≠未注册；复制 runs-on 不写入仓库。

## D. 主机、实例与服务（C3 / C5 / D3）

- [ ] D01 machine_id 重启/hostname 改名后不变；目录和 registry 使用 user/repo/machine 身份，不依赖仓库名。
- [ ] D02 两个真实测试仓库各有独立实例，互不覆盖；未知 Runner 或 registry 丢失不得凭名称接管/删除。
- [ ] D03 注册→标签→启动→双侧回读；点击两次不重复注册；不使用 --replace 覆盖外部实例。
- [ ] D04 Active/Standby/Disabled/Quarantined 后置条件与 shared-contracts 一致；标签移除后服务仍运行不能标已停用。
- [ ] D05 已知 busy 时普通切换/停止/注销拒绝；紧急停止另行确认并提醒中断风险。
- [ ] D06 idle→新作业接单竞态有受控测试，结果不声称无损排空；无法停止时不得继续标签变更或注销。
- [ ] D07 应用关闭/退出登录不默默停止既有 Runner；服务手动启动，电脑重启/GRC 重开不自动恢复。
- [ ] D08 离线/未登录也能停止明确归属的本机实例，远端 Unknown 不伪装成删除/清理完成。
- [ ] D09 注销确认双侧状态；残留单独报告。路径穿越、符号链接/reparse point 越界和其他实例目录均不被清理。

## E. Apply 与恢复（C4 / D2 / D3）

- [ ] E01 Preview 绑定账户/session、repo/machine/runner、策略与快照；编辑后、远端被外部修改后必须重预览。
- [ ] E02 无差异零 API 写、零重启；重复 Apply 不重放已经完成的授权动作。
- [ ] E03 远端部分标签更新后故障，保留 Partial 与真实 observed；不自动回滚或重启。
- [ ] E04 注册请求响应丢失/进程崩溃，重开先核对，不能制造第二个实例；原操作日志可定位。
- [ ] E05 Reconcile 只读；恢复写入重新确认。404 区分实例不存在、标签不存在和无权限/不可见，不统一当成功。
- [ ] E06 GET 重试/等待有上限且遵守 Retry-After；写操作不盲重试，异常不变成无限后台任务。
- [ ] E07 普通授权动作在审计不可写时拒绝，紧急停止仍执行并明示日志失败；损坏配置不覆写为空配置。

## F. 安全边界（C2 / C3 / C4 / D3）

- [ ] F01 OAuth scope 广度明示；scope/仓库权限不足不自动提权；无 client secret、任意项目脚本执行或 localhost 管理服务。
- [ ] F02 Runner 低权限身份不同于控制器用户，不以管理员/LocalSystem 执行业务作业；提升流程只可管理受控 GRC 实例。
- [ ] F03 从真实 Runner 身份读取控制器凭据/registry/日志、修改控制器程序/策略的受控探针全部被拒绝；不以“目录不同”代替 ACL 证据。
- [ ] F04 私人仓库仍需上述边界；公共仓库缺专用执行环境声明/确认时启用拒绝。声明仅 OperatorAttested，不宣传自动证明隔离。
- [ ] F05 官方 Runner 来源/版本/校验摘要可追踪，校验失败不执行；注册/移除 token 不落日志/历史，OAuth 不进入 Runner 环境。
- [ ] F06 异常、HTTP、进程、诊断复制、审计均无秘密；fixture 不含真实 token。扫描通过不是绝对无泄漏保证。
- [ ] F07 审计轮转有界，未完成操作记录不随轮转消失；本机日志不标为防篡改或永久全历史。

## G. 真实端到端与打包（C6 / D4）

- [ ] G01 在用户授权的两个非 Utopia 测试仓库、至少一台真实 Windows 上完成桌面登录、完整列表、独立注册、角色变更、停止/启动、注销和审计。
- [ ] G02 用户在测试仓库自行放置受信任的最小 workflow，并用 GRC 给出的角色标签路由；实际作业返回 runner name、OS、测试 commit SHA，确认跑在指定本机实例，而不是只看到标签写入。
- [ ] G03 停止后实例不再执行测试作业；未匹配/离线时排队而非自动云端回退，取消测试队列由用户操作，不添加调度功能。
- [ ] G04 至少展示一次 busy 拒绝、一次 stale/部分失败及恢复、一次退出重开；保留失败和修复记录，不只留成功截图。
- [ ] G05 合并后的同一完整 SHA 完成 restore/build/test 和 Windows 打包验证；记录 Windows/.NET/Avalonia/Runner 版本、包哈希与复现步骤。
- [ ] G06 Codex 复核未主导的 UI，DeepSeek 复核未主导的核心。记录真实 host/model；同一执行者全部完成只能报自检。

## Definition of Done

A–G 必需项全部 PASS，且真实 Windows/GitHub 作业证据完整，才可标 `PHASE1_MANAGER_COMPLETE`。缺外部凭据/环境为 BLOCKED，对应 NOT_RUN；只有 fake/单元测试为 PARTIAL。普通单元测试本就允许 mock，不要求每个异常都破坏真实 GitHub 环境复现。

完整交付不要求第二台物理机、Linux/macOS、VM 编排、独立性裁决、项目生命周期或任何非目标功能。最终结论只覆盖本版管理器，不升级为安全认证或研究平台验收。
