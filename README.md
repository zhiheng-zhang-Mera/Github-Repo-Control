# Github-Repo-Control

Github-Repo-Control 是一个独立于任何业务项目的 **GitHub 全局机器控制器**。

它不属于 Utopia、PCF、Digital-City 或任何被管理仓库，也不接受这些仓库的反向控制。它的职责是：在用户登录 GitHub 后，列出当前账户授权范围内的仓库，并允许用户从桌面应用中手动管理“当前主机”在每个仓库中的 self-hosted runner 注册与角色。

## 第一阶段目标

第一阶段只做一条完整、可验证的主链。仓库列表以 GitHub 当前远端状态为真值：每次打开应用必须主动刷新一次，用户也可以随时手动刷新；本地缓存不得被当作最新列表。

1. 启动桌面应用。
2. 登录 GitHub。
3. 每次启动应用都重新向 GitHub 拉取当前账户授权范围内的全部仓库；仓库列表页同时提供手动“刷新”按钮。
4. 选择一个仓库。
5. 检测当前主机是否已为该仓库注册 self-hosted runner。
6. 用户为当前主机选择一个或多个兼容角色。
7. 本地冲突引擎先检查角色组合。
8. 用户确认后才应用变更。
9. 将角色同步为 GitHub runner custom labels，并展示最终状态。
10. 所有注册、取消注册、角色变更都写入本地审计日志。

## 项目定位

本项目是账户/机器级基础设施控制面，而不是业务项目的一部分。

```text
GitHub Account
      │
      ▼
Github-Repo-Control
      │
      ├── Repo A ── Current Host: CI + Build
      ├── Repo B ── Current Host: General
      └── Repo C ── Current Host: Verify

被管理仓库不能反向调用或修改 Github-Repo-Control。
```

## 桌面技术方向

首版采用跨平台桌面架构：

- .NET LTS
- Avalonia UI
- MVVM
- GitHub REST API
- 本地安全凭据存储
- 本地结构化审计日志
- Windows 为第一验收平台，同时保持 Linux/macOS 可移植性

依赖版本由施工时选择当前稳定版本并锁定，不在设计文档中硬编码短生命周期版本号。

## 角色模型

角色属于 **“当前主机 × 仓库”**，而不是主机的全局唯一身份。

首版预置：

- `general`：一般机器，可承担项目定义的普通 self-hosted 工作。
- `ci`：CI 测试。
- `build`：构建。
- `dev`：开发/施工。
- `verify`：独立验证。
- `repair`：修补。
- `platform-test`：平台专项测试。

允许多角色并存，例如：

```text
general + ci
ci + build
general + platform-test
```

但存在硬冲突时必须拒绝应用。例如同一仓库中，为保持独立验证语义：

```text
verify × dev       = conflict
verify × repair    = conflict
```

更完整的兼容矩阵见 [design/role-model.md](design/role-model.md)。

> `standby`、`disabled`、`quarantined` 属于运行状态，不作为可叠加角色。

## 安全边界

以下约束为硬规则：

- 被管理仓库内容视为不可信输入。
- 不执行仓库中的脚本、配置或二进制文件来决定控制器权限。
- 不允许 GitHub Actions workflow 调用控制器完成提权或角色切换。
- 所有角色修改必须由本机用户明确触发。
- GitHub token 不得写入仓库、配置文件或日志。
- 凭据必须进入系统安全凭据存储。
- runner 实例与控制器凭据分离。
- 每个 repository-level runner 使用独立实例目录，避免多仓库配置互相覆盖。
- 控制器必须记录审计日志，但日志中不得记录 access token、registration token 等秘密。

## 目录

```text
Github-Repo-Control/
├── README.md
└── design/
    ├── README.md
    ├── architecture.md
    ├── role-model.md
    ├── acceptance.md
    ├── CODEX-WORKBOOK.md
    └── DEEPSEEK-WORKBOOK.md
```

当前 `design/` 为施工前的规范基线。Codex 与 DeepSeek 的任务必须以其中的共享契约为准，不得自行放宽安全边界或修改角色冲突语义。

## 当前阶段

状态：**DESIGN / READY_FOR_IMPLEMENTATION**

下一步由 Codex 和 DeepSeek 按 `design/` 中的施工书实施，完成后再进入独立验收与桌面打包阶段。
