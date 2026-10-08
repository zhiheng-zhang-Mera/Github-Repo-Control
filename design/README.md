# design 设计索引

本目录定义 Github-Repo-Control 第一阶段的施工基线。

## 目标

Github-Repo-Control 是独立桌面控制面。它登录 GitHub 后读取当前账户授权范围内的仓库，并允许用户手动设置“当前主机”在各仓库中的 self-hosted runner 注册状态与角色。

第一阶段只管理 **当前主机**，不做远程控制其它主机。

## 文档

- [architecture.md](architecture.md)：系统架构、数据流、安全边界、runner 生命周期。
- [role-model.md](role-model.md)：角色、状态、兼容矩阵、冲突检测。
- [acceptance.md](acceptance.md)：共同验收标准。
- [CODEX-WORKBOOK.md](CODEX-WORKBOOK.md)：Codex 核心施工线。
- [DEEPSEEK-WORKBOOK.md](DEEPSEEK-WORKBOOK.md)：DeepSeek UI/独立验证施工线。

## 不可破坏的共享契约

1. 桌面应用是唯一控制入口；普通被管理仓库不能反向修改控制器状态。
2. 被管理 repo 的源码、workflow、配置文件全部视为不可信输入。
3. 角色变更必须经过本地冲突检测，并由用户明确点击“应用”。
4. 角色属于 `machine × repository`。
5. 支持多个兼容角色同时存在。
6. 固定硬件能力与动态项目角色分离。
7. `standby / disabled / quarantined` 是状态，不是角色。
8. repository-level runner 每个 repo 使用独立 runner instance，不能复用同一个配置目录去覆盖注册。
9. GitHub token 和 runner registration token 不进入 repo、不进入日志。
10. Codex 与 DeepSeek 不得通过修改这些规则来规避验收。

## 推荐开发顺序

```text
共享模型与冲突引擎
        ↓
GitHub 登录与仓库枚举
        ↓
当前主机识别
        ↓
runner 状态发现
        ↓
角色编辑 UI
        ↓
注册 / label 同步
        ↓
审计日志
        ↓
故障恢复与验收
```
