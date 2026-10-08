# design 设计与施工索引

**版本：GRC-MANAGER-V1.1 · 2026-10-08**

本轮仅修订设计和施工书，不启动产品施工，不标记运行验收完成。修订前基线为 `2b906b223276bcaa342599da0e22456b013281a2`；历史原文由 Git 保留，不复制另一套仍可能被误用的规范。

## 一、固定目标

GRC 只管理“当前主机在每个 GitHub 仓库中的 Runner 与角色”。新项目出生后主动刷新并手动配置即可，不引入 Project Onboarding 子系统，不控制项目的研究决策、普通开发权限或代码提交。

Utopia 已冻结，不作为施工依赖、测试仓库或修改目标；新项目独立存在，GRC 不内嵌进去。

## 二、权威文档

| 文件 | 权威范围 |
|---|---|
| [architecture.md](architecture.md) | 功能边界、权限、安全、生命周期与恢复 |
| [role-model.md](role-model.md) | 角色集合、冲突、不变量 |
| [shared-contracts.md](shared-contracts.md) | UI/Core 公共类型、服务签名和错误语义 |
| [acceptance.md](acceptance.md) | 验收用例与证据要求 |
| [CODEX-WORKBOOK.md](CODEX-WORKBOOK.md) | C0–C6 核心施工及交付 |
| [DEEPSEEK-WORKBOOK.md](DEEPSEEK-WORKBOOK.md) | D0–D4 界面施工与整体验证 |

施工书不得覆盖规范；出现冲突须记录具体条款与影响，不得通过降低安全规则或改状态名称来“过验收”。新增非目标功能先记建议，不纳入这一版交付。

## 三、最短施工路线

```text
C0：工程骨架 + 共享契约 + fake（一个明确提交）
   ├── Codex：C1 → C2 → C3 → C4 → C5
   └── DeepSeek：D0 → D1 → D2（只消费共享契约与 fake）
核心交付 C6 + UI 交付 D2
   → D3：明确 integration SHA，双方各复检未主导部分
   → D4：真实 Windows 桌面 + GitHub Runner 作业端到端
   → 一次集中修补 + 定向确认 → COMPLETE 或如实 PARTIAL/BLOCKED
```

先交付契约，不等待所有后端完成才让 UI 开始；UI 也不能复制另一套后端解阻。C0 后 App 归 DeepSeek，Core/GitHub/Runner/Security 归 Codex；共享契约改动由双方显式同步。角色名称不绑定真实模型能力，单人顺序执行同样允许，但不能自称独立复核。

每次开工记录 `hostname + model + worker_id + base_sha + spec_sha`，均以实际完整 SHA 为准。分支名仅作导航。最终报告锚定 integration SHA，不把两个未合并分支分别通过测试当作完整交付。

缺 OAuth client id、管理员预置执行身份或真实验收仓库时，仅阻塞相关真实步骤；可继续契约、fake、UI 与单元测试。不得无限轮询、绕过授权、虚构通过或反复制造同一种验收循环。

## 四、本轮强化与主动减法

保留：完整仓库刷新、全部角色可见、兼容多选、用户确认、当前主机管理、审计、中文主页。

修补：标签≠权限、角色≠独立性；忙碌切换及停止竞态；部分成功和崩溃恢复；账户/仓库 ID 绑定；控制器与执行身份隔离；公共仓库启用风险；UI/Core 契约先行；真实作业不只看标签。

明确不做：项目创建/审批/准入引擎、远程主机控制、智能调度、自动改 workflow、权限/分支规则管理、隔离环境编排、论文数据平台和 Utopia 后续更新。

详细 GitHub 依据统一收在 [架构的官方依据](architecture.md#十二官方依据)，施工时记录实际 API 与依赖版本，不从旧对话推定平台能力。
