# 架构设计

## 1. 定位

Github-Repo-Control 是 GitHub 账户级的本地桌面控制器。

它不属于任何被管理项目。Utopia、PCF、Digital-City 以及未来项目都只能作为普通 repository 出现在它的仓库列表中。

```text
GitHub
  │
  ▼
Github-Repo-Control
  ├── Authentication
  ├── Repository Discovery
  ├── Current Host Identity
  ├── Runner Instance Manager
  ├── Role Conflict Engine
  ├── GitHub Runner API Adapter
  └── Audit Log
        │
        ▼
Repository-level self-hosted runner instances
```

## 2. 技术栈

首版：

- .NET LTS
- Avalonia UI
- MVVM
- typed HttpClient
- GitHub REST API
- JSON 本地配置 + 结构化审计日志
- 系统安全凭据存储

不得把 token 存入普通 JSON、SQLite 明文、repo 文件或日志。

Windows 为首要验收平台；接口层必须保留 Linux/macOS 实现边界。

## 3. 登录

首选 GitHub App / Device Flow。

桌面应用启动时：

1. 若无有效登录态，显示“登录 GitHub”。
2. 使用 Device Flow 打开 GitHub 验证。
3. 获取用户授权后读取账户与可访问 repository。
4. token 写入系统安全凭据存储。
5. UI 显示当前登录账户、授权范围与权限不足的仓库。

登录模块必须允许 logout，并在 logout 后清除本地凭据和内存中的 token。

如果当前 GitHub App 安装只授权部分仓库，UI 必须明确写“已授权仓库”，不能伪装成账户全部仓库。

## 4. 仓库枚举

主界面显示：

- owner/name
- visibility
- archived
- 当前用户对 repo 的权限
- 当前主机是否注册 runner
- 当前主机在该 repo 的角色摘要
- runner online/offline/busy 状态（可获得时）

支持：

- 搜索
- owner 过滤
- public/private 过滤
- 已注册/未注册过滤
- 角色过滤

不 clone repo。

允许读取 workflow/runner metadata 用于展示，但不得执行仓库内容。

## 5. 当前主机身份

本机 identity 与 repo 无关。

建议字段：

```text
machine_id      本地生成并持久化的 UUID
hostname
os
architecture
app_instance_id
capabilities
```

`machine_id` 第一次运行生成，之后保持稳定；hostname 变化不能导致误认为另一台机器。

硬件/平台 capability 与动态 role 分离。

示例 capability：

- windows
- linux
- macos
- x64
- arm64

后续可扩展 gpu/android/ios，但第一阶段不得为了探测能力执行被管理项目中的代码。

## 6. Runner 实例

### 6.1 repository-level runner

第一阶段按 repository-level runner 设计。

同一主机可以服务多个 repo，但每个 repo 必须有独立实例目录：

```text
<AppData>/Github-Repo-Control/runners/
  owner__repoA/
  owner__repoB/
  owner__repoC/
```

禁止用同一 runner 配置目录反复覆盖注册。

### 6.2 实例状态

每个 repo 的当前主机 assignment 至少记录：

```text
NotRegistered
Registering
RegisteredOffline
RegisteredOnline
Busy
Stopping
Error
```

任何失败必须进入可恢复状态，并在 UI 呈现错误原因。

### 6.3 注册

用户第一次给未注册 repo 应用有效角色时：

1. 权限预检。
2. 请求 repository runner registration token。
3. 创建独立 runner instance。
4. 注册当前主机。
5. 设置 custom labels。
6. 启动 runner。
7. 二次读取 GitHub 状态确认。
8. 仅确认成功后更新本地 assignment。

registration token 为短生命周期秘密，只驻留内存，禁止写日志。

## 7. Role 与 label

自定义 label 使用固定 namespace，避免与项目自行定义的 label 混淆：

```text
grc-role-general
grc-role-ci
grc-role-build
grc-role-dev
grc-role-verify
grc-role-repair
grc-role-platform-test
```

系统/能力 label 不由角色编辑器任意篡改。

Apply 时只修改 `grc-role-*` 集合，不得无差别覆盖 GitHub 返回的其它 custom labels。

## 8. 手动控制原则

角色变更只能来自：

```text
Local User
   ↓
Desktop UI
   ↓
Conflict Engine
   ↓
Confirmation
   ↓
Runner API / Local Runner Manager
```

明确禁止：

```text
Managed Repo
  ↓
workflow / script / config
  ↓
change controller role
```

不提供普通 localhost HTTP 管理端口给 repo 调用。

如果未来必须提供 IPC，则必须使用 OS ACL、独立身份验证与用户确认；第一阶段不做。

## 9. 应用角色变更

Apply 使用“计划-确认-执行-核验”事务语义：

1. 读取最新远端 runner 状态。
2. 读取当前角色。
3. 计算 desired roles。
4. Conflict Engine 验证。
5. 生成 diff。
6. UI 显示 diff。
7. 用户确认。
8. 应用。
9. 再读 GitHub 状态。
10. 成功才写 audit success；失败写 audit failure 并显示恢复操作。

不得采用“UI 先显示成功、后台再慢慢同步”的乐观假成功。

## 10. 审计日志

至少记录：

- timestamp
- machine_id
- hostname
- repository
- action
- before roles
- after roles
- result
- error code / sanitized error

禁止记录：

- access token
- refresh token
- device code
- registration token
- Authorization header

## 11. 控制器自身隔离

控制器数据目录必须位于项目 workspace 之外。

runner 工作账户原则上不应拥有读取控制器凭据存储的权限。

首版至少做到：

- controller secrets 不出现在 runner work directory。
- controller 不接受 runner workflow 的命令。
- repo 不可通过提交文件改变全局 role policy。
- role policy 编译进控制器或来自本控制器自己的受信配置。

## 12. 非目标

第一阶段明确不做：

- 远程操作其它主机。
- 自动根据 repo 配置提升角色。
- Utopia/PCF 集成。
- 自动任务调度优化。
- 跨账户多租户。
- 容器/Kubernetes 编排。
- 自动执行 repo 中的 setup 脚本。
