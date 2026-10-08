# 第一阶段验收标准

## A. 启动与登录

- [ ] Windows 桌面应用可直接启动。
- [ ] 未登录时只能进入登录流程，不出现伪造 repo 数据。
- [ ] GitHub Device Flow 成功。
- [ ] 登录完成显示真实 GitHub 账户。
- [ ] logout 后 token 从安全存储与进程内清除。
- [ ] token、device code、registration token 不出现在普通日志。

## B. Repository 列表

- [ ] 每次冷启动应用并恢复有效登录态后，自动向 GitHub 发起一次 repository 全量分页刷新。
- [ ] 每次新登录成功后自动刷新一次 repository 列表。
- [ ] Repository List 页面存在显式“刷新”按钮。
- [ ] 手动刷新实际调用 GitHub API，不得只重载本地缓存。
- [ ] 刷新时 UI 有明确 loading/refreshing 状态且不冻结。
- [ ] 刷新失败时允许显示旧缓存，但必须标记 stale，并显示最后成功刷新时间。
- [ ] 刷新成功后新增/删除/权限变化的 repo 能正确反映。
- [ ] 能分页读取当前授权范围内 repo。
- [ ] public/private 都能正确显示。
- [ ] archived 状态正确。
- [ ] 支持搜索与基本过滤。
- [ ] API 失败时有明确错误和重试，不显示旧数据为“最新”。
- [ ] 不 clone repo。

## C. 当前主机

- [ ] 第一次运行生成持久 machine_id。
- [ ] 显示 hostname / OS / arch。
- [ ] 重启应用后 machine_id 不改变。
- [ ] hostname 改变时不会自动生成新 machine_id。

## D. Runner

- [ ] 选择 repo 后能发现当前主机 runner assignment。
- [ ] 未注册 repo 可由用户明确操作后注册。
- [ ] 每个 repo 的 runner 使用独立目录。
- [ ] 注册失败可恢复，不留下“UI 显示成功但 GitHub 不存在”的状态。
- [ ] 已注册 runner 可启动/停止。
- [ ] 移除注册必须二次确认。
- [ ] registration token 不落盘。

## E. 角色

- [ ] 支持多选。
- [ ] `general + ci` 可应用。
- [ ] `ci + build` 可应用。
- [ ] `dev + verify` 被硬拒绝。
- [ ] `repair + verify` 被硬拒绝。
- [ ] 冲突必须在调用 GitHub 写 API 之前发现。
- [ ] 角色最终映射为 `grc-role-*` labels。
- [ ] Apply 不得删除不属于本控制器 namespace 的其它 labels。
- [ ] 远端状态被其他端修改后，本端不能静默覆盖。

## F. 安全隔离

- [ ] 被管理 repo 无法通过文件内容改变 role policy。
- [ ] 不执行 repo 中任何脚本。
- [ ] workflow 无公开 localhost 控制 API 可直接调。
- [ ] 凭据不在 runner workspace。
- [ ] 审计日志已脱敏。
- [ ] 对权限不足 repo 只显示可解释状态，不循环请求 admin API。

## G. UI

最少包含：

1. Login 页面。
2. Repository List 页面。
3. Repository Detail / Current Host 页面。
4. Role multi-select。
5. 冲突提示。
6. Apply diff 确认。
7. Runner 状态。
8. Audit 页面或可读审计视图。

## H. 测试门槛

- [ ] RoleConflictEngine 单元测试覆盖全部矩阵组合。
- [ ] GitHub API adapter 有 mock 测试。
- [ ] token redaction 测试。
- [ ] runner path isolation 测试。
- [ ] Apply stale-state/optimistic concurrency 测试。
- [ ] Windows 至少一次真实账户端到端测试。
- [ ] 测试 fixture 不包含真实 token。

## I. Definition of Done

只有同时满足以下条件才能标记 Phase 1 COMPLETE：

```text
Login
+ Repo Discovery
+ Current Host Identity
+ Runner Registration
+ Multi-role Editing
+ Conflict Rejection
+ Label Synchronization
+ Audit
+ Security Boundary
+ Real E2E Evidence
```

任何一项以 mock 代替真实端到端验证时，只能标记 PARTIAL。
