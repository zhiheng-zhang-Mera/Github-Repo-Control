# Github-Repo-Control

Github-Repo-Control（GRC）是独立的 **GitHub 仓库 × 当前主机 Runner 管理器**，不是研发平台、调度器或项目治理系统。

登录 GitHub 后，用户查看当前授权范围内的仓库，手动管理当前主机在各仓库中的 Runner 注册、运行状态与预定义角色。它不属于 Utopia、PCF、Digital-City，也不接受被管理项目的反向命令。

## 范围与当前阶段

- **设计版本：GRC-MANAGER-V1.1（2026-10-08）。** 状态仍为 `DESIGN / READY_FOR_IMPLEMENTATION`，不代表桌面程序已经实现或通过验收。
- Utopia 此后冻结；本项目的施工、验证与展示不得修改 Utopia，也不要求 Utopia 接入或继续更新。
- 新孵化项目创建后，通过“刷新发现 → 选择仓库 → 配置当前主机 → 确认应用”接入。没有独立的项目准入审批、创建阻断或生命周期引擎。
- GRC 不拦截普通 Git clone/push，不授予 GitHub 仓库写权限，不决定谁可以开发。未注册 Runner 的普通电脑仍可正常开发。
- 第一阶段：GitHub.com、单个当前登录账户、当前 Windows 主机、repository-level Runner；保留 Linux/macOS 接口，不宣称已支持。

## 一条主链

```text
启动 / 登录 GitHub
  → 启动时完整分页刷新仓库（也可随时手动刷新）
  → 选择仓库，读取当前主机与 Runner 实况
  → 从完整角色列表勾选兼容角色
  → 预览变更，用户确认
  → 注册或更新当前主机 Runner
  → 回读 GitHub + 本机状态，记录脱敏结果
```

新发现仓库默认未托管，不批量注册、不静默启动。列表刷新失败可以显示旧缓存，但必须标记过期；部分分页结果不能冒充全量最新列表。Runner 细节按需读取，不在启动时为每个仓库重复调用管理 API。

## 角色与状态

角色属于 `当前主机 × 仓库`，支持 `general / ci / build / dev / verify / repair / platform-test`。全部通过 checkbox/toggle 列表选择；禁止自由文本角色、任意 label 编辑或隐藏的自定义角色入口。

`verify` 与 `dev`、`repair` 硬冲突：冲突项保留显示但立即禁用，Core 再验证。其他既有兼容组合保留，例如 `general + ci`、`ci + build`。运行状态 `Active / Standby / Disabled / Quarantined` 单独设置，不算角色；隔离仅是停止与本地安全锁，不包含威胁检测平台。

**角色标签只是作业路由元数据，不是安全权限或独立验证证明。** `verify` 不保证物理主机、模型或任务历史独立；GRC 不追踪这些研究语义。移除标签不等于停止接单，停用必须确认本机 Runner 已停止。详见 [角色模型](design/role-model.md)。

## 日常使用边界

- 忙碌时拒绝普通角色切换、停止和注销；用户可另行确认紧急停止，界面明确任务可能中断。首版不做排空队列或自动等待后执行。
- 关窗口或退出 GitHub 登录，不会默默停止已批准运行的 Runner；退出登录会清除控制器登录态。停止是独立操作，离线时也可停止已核实归属的本机 Runner。
- 首版服务启动类型为手动；重启电脑或重新打开 GRC 不自动恢复运行，需用户再次启动。
- 不自动编辑 workflow。详情页只提供可复制的 `runs-on` 片段和 GitHub 设置入口；是否采用由各项目决定。

## 最低安全要求

控制器凭据进入系统安全存储，不进入仓库、Runner 工作区、日志或 Runner 环境变量。Runner 使用不同于控制器用户的低权限执行身份；目录分开仅避免配置覆盖，不等于安全沙箱。权限边界未建立时只能浏览，不能启用执行。

首版只支持受信任工作流。公共仓库默认禁止启用自托管执行；只有用户在外部准备专用执行主机或隔离虚拟机、明确确认用途和风险，并通过本机身份/ACL 检查后，才允许显式启用。GRC 不创建虚拟机，也不认证工作流安全；专用环境声明是用户声明，不是假装自动验证。私人仓库同样不能免除执行身份隔离。

## 桌面实现与施工入口

采用 .NET LTS + Avalonia + MVVM + GitHub REST API。依赖与 Runner 版本在施工时选定、锁定并记录。界面只需登录、仓库列表、仓库详情、审计视图，账户设置用轻量对话框。

| 文档 | 内容 |
|---|---|
| [设计索引](design/README.md) | 范围、施工次序、修订说明 |
| [架构](design/architecture.md) | 登录、状态、生命周期、安全、故障恢复 |
| [角色模型](design/role-model.md) | 完整目录、冲突矩阵、标签语义 |
| [共享契约](design/shared-contracts.md) | 核心/UI 接口与统一错误语义 |
| [验收清单](design/acceptance.md) | 可核查的完成标准 |
| [Codex 施工书](design/CODEX-WORKBOOK.md) | 核心、GitHub 与本机 Runner |
| [DeepSeek 施工书](design/DEEPSEEK-WORKBOOK.md) | UI、集成与交叉验证 |

不开发远程控机、全局算力调度、多账户并行、GitHub 权限治理、自动 PR/合并、云端回退、账单优化、项目创建向导、插件市场或 Utopia 集成。
