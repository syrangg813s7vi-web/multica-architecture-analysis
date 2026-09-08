---
layout: default
title: Multica 全仓库架构分析
permalink: /analysis/
---

# Multica 全仓库架构分析

> 分析基线：`c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd`（2026-08-28）。本报告以当前检出的源码、迁移、测试、部署配置和仓库文档为证据。它是全仓库静态架构分析，不等价于逐行业务正确性证明或生产渗透测试。

## 1. 结论摘要

Multica 不是“另一个 ChatGPT UI”，而是一个面向软件团队的 **Agent 控制平面 + 分布式执行平面**：服务端保存项目、Issue、Chat、Agent、Skill、Autopilot 和任务状态；安装在开发机或服务器上的 daemon 领取任务、准备 Git 工作区和 Agent 配置，再启动外部 Agent CLI。Web、Desktop、Mobile、IM Channel、Webhook 和插件是不同入口，最终汇入同一任务与协作模型。

它已经具备较完整的多 Agent 工程底座：持久任务队列、远端 daemon、多个 Agent Runtime、任务级 Token、Git worktree、会话恢复、Skills、MCP、Autopilot、渠道接入和多 VCS Provider。最明显的能力边界有三项：

1. daemon **不是安全沙箱**，任务可继承 daemon 系统用户能访问的文件与凭据；强隔离必须由专用用户、容器或 VM 提供。
2. 内置自托管部署 **没有完整灾备闭环**；数据库、上传对象、密钥清单、RPO/RTO 和恢复演练需要部署方补齐。
3. VCS 与 Runtime 都是“有扩展点但非任意兼容”。公司内部 CodeHub 若不能完整模拟现有 GitLab/Gitea/Forgejo 接口，就需要新增 Provider 适配，而不仅是改一条 clone 命令。

对应图：

- [系统全景架构](../diagrams/system-architecture.html)
- [Agent 任务执行时序](../diagrams/agent-task-execution.html)
- [任务状态生命周期](../diagrams/task-lifecycle.html)

## 2. 仓库规模与交付单元

仓库采用 Go 后端和 pnpm/Turborepo 前端单体仓库。约有 2604 个 `server` 文件、1719 个 `packages` 文件、898 个 `apps` 文件；Go/TS/TSX 总计约 107 万行（包含生成代码与测试）。测试资产约 888 个 Go 测试文件和 650 个 TS/TSX 测试文件。

| 交付单元 | 技术与职责 | 主要入口 |
|---|---|---|
| API Server | Go HTTP/WebSocket、调度器、领域服务、集成入口 | [server/cmd/server/main.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/cmd/server/main.go) |
| Multica CLI / Daemon | 登录、机器注册、任务领取、执行环境与 Agent 进程控制 | [server/cmd/multica](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/cmd/multica)、[server/internal/daemon/daemon.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/daemon.go) |
| Web | Next.js 外壳，组合共享 `views/core/ui` | [apps/web](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/apps/web) |
| Desktop | Electron main/preload/renderer，额外管理 daemon、CLI、更新与本机能力 | [apps/desktop/src/main](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/apps/desktop/src/main) |
| Mobile | Expo Router 独立客户端，包含自己的 API、状态、Realtime 与移动 UI | [apps/mobile](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/apps/mobile) |
| Docs | Fumadocs 文档站 | [apps/docs](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/apps/docs) |
| Shared packages | `core` 数据/API/状态、`ui` 组件、`views` 业务页面、Plugin SDK | [packages](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/packages) |
| Migration tools | 主迁移器和多个历史数据 backfill 工具 | [server/cmd/migrate](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/cmd/migrate) |

仓库迁移文件约 938 个（up/down 成对，编号已到 440 左右），说明产品迭代密集、领域面广。最大的生产文件包括 daemon 主体约 9.2k 行、TaskService 约 7.3k 行、daemon handler 约 5.2k 行、前端 API client 约 4.7k 行和 issue handler 约 4.4k 行，是后续维护与拆分的重点。

## 3. 总体运行架构

### 3.1 控制平面

API Server 在启动时组装 PostgreSQL、可选 Redis、领域服务、进程内事件总线、用户 Realtime Hub、daemon WebSocket Hub、Scheduler、存储、认证缓存、指标、Analytics、Entitlements 和 Channel 服务。组装入口集中在 [server/cmd/server/main.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/cmd/server/main.go)，Handler 聚合入口在 [server/internal/handler/handler.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/handler/handler.go)。

控制平面的职责是：

- 保存 Workspace、Project、Issue、Chat、Agent、Runtime、Skill、Plugin、Autopilot 和审计状态；
- 将人工分配、评论提及、Squad、Chat、Channel、Autopilot、Quick Create 等触发统一转成 Agent Task；
- 校验 Workspace、Agent、Runtime、仓库和插件权限；
- 向 daemon 派发任务，并接收消息、使用量、结果和终态；
- 通过 WebSocket 或外部 Channel 把状态反馈给用户。

### 3.2 执行平面

Daemon 是机器级执行控制器。它长连接服务端，发送 heartbeat、领取任务、下载上下文和 Skill，建立任务目录，准备 Runtime Home/MCP 配置，启动 Agent CLI，流式解析结构化消息，并负责取消、进程树清理、会话复用和环境 GC。

核心实现集中在 [server/internal/daemon/daemon.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/daemon.go)、[server/internal/daemon/execenv/execenv.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/execenv/execenv.go) 和 [server/pkg/agent](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent)。服务端 Claim 构造和租约落库集中在 [server/internal/handler/daemon.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/handler/daemon.go)。

控制平面与执行平面分开带来两个直接收益：服务端不需要直接 SSH 到每台机器；同一个 Multica 服务可以连接多台 Linux/macOS/Windows 执行机。代价是运行可靠性依赖 heartbeat、租约、重连宽限、任务 Token 和 orphan recovery 的一致协作。

## 4. 服务端分层与核心领域

服务端大体遵循以下单向依赖：

```text
cmd/server → handler + middleware → service → db/sqlc + domain packages
                                      ↓
                              events / realtime / integrations
```

- `handler` 负责 HTTP 契约、输入输出和权限边界，不应承载完整业务状态机。
- `service` 负责事务、领域规则和跨模块编排。
- `pkg/db/queries` 是 SQL 事实源，`pkg/db/generated` 是 sqlc 生成层。
- `integrations` 把第三方协议转换为内部命令或事件。
- `daemonws` 和 `realtime` 分别服务机器连接与用户连接。

数据库约有一百张表，可归为八组：

1. Workspace、成员、权限、邀请与计费；
2. Project、Issue、自定义状态/属性、评论、附件与 Source Context；
3. Agent、Runtime、机器、Runtime Profile 和 Agent Skill 绑定；
4. `agent_task_queue`、消息、Usage、Trace、会话与失败分类；
5. Chat、Channel 会话、绑定、消息投递与媒体账本；
6. Autopilot、版本化 Runbook、Trigger、Run、Subscriber/Collaborator；
7. Plugin、版本、安装、Grant、Secret、Hook、Action/MCP 调用；
8. GitHub/GitLab/Gitea/Forgejo 等 VCS 连接与外部对象映射。

数据库刻意不使用外键与级联删除，把兼容性和清理逻辑放在服务与迁移中。这减少了在线迁移阻力，但也意味着引用完整性、删除顺序和孤儿清理更依赖应用测试。

## 5. Agent Task 的完整链路

### 5.1 创建与排队

[server/internal/service/task.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/service/task.go) 是最关键的领域编排热点。任务可由 Issue 指派、评论 @Agent、Squad Leader、Chat、Channel、Autopilot 或 Quick Create 产生，统一写入 `agent_task_queue`。数据库而不是 WebSocket 是跨进程事实源；Realtime 只负责加速通知。

### 5.2 Claim 与权限收敛

在线 daemon heartbeat 后按 Runtime 领取任务。服务端在 Claim 阶段重新校验：

- Agent 与 Runtime 是否匹配；
- Workspace 与来源对象是否一致；
- Project 允许访问哪些仓库或本地目录；
- Runtime Profile、模型、思考强度与连接应用；
- Skill bundle、内置 Skill、Issue/Chat/Autopilot 上下文；
- 本次任务需要的短期凭据。

Claim 成功后创建绑定 Agent/Task 的 `mat_` 任务 Token。它与 daemon 凭据、人类 PAT/JWT 分离；任务取消时撤销，避免 Agent 以 daemon owner 的身份扩大权限。相关鉴权入口在 [server/internal/middleware/auth.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/middleware/auth.go) 和 daemon Claim Handler。

### 5.3 环境准备与 Runtime

任务环境包含：

- Git checkout 或 local directory；
- Issue/Chat/Autopilot 上下文文件；
- 持久或临时 Skills；
- MCP server 配置和连接应用 overlay；
- Runtime 专属 HOME（如任务级 `CODEX_HOME`）；
- 会话恢复信息、生命周期锁和 GC 标记。

Runtime 适配层同时包含厂商专有 CLI/NDJSON 协议和 ACP JSON-RPC 风格协议，统一处理进程启动、流式消息、usage、模型目录、thinking/service tier、resume 和 cancel。新公司 Agent CLI 最低摩擦的接法不是修改整个任务系统，而是让 CLI 兼容现有协议族或新增一个局部 Runtime Adapter。

### 5.4 状态与恢复

主要状态是：

```text
queued → dispatched → running → completed | failed
                    ↘ waiting_local_directory
任意活动阶段 → cancelled
```

此外，带 `fire_at` 的重试可处于 `deferred`。系统包含 prepare lease、task lease、排队 TTL、daemon reconnect grace、取消 ack、orphan recovery、session pin/resume 和结构化失败原因。重跑通常创建一个新的 queued attempt，保留原失败记录，而不是覆写历史。

## 6. Git、Worktree 与并行开发

远程仓库由 [server/internal/daemon/repocache/cache.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/repocache/cache.go) 维护 bare cache，再为任务创建 linked worktree；特定平台/Runtime 可改用隔离的本地 Git metadata 或 local clone，以规避 Linux/Windows 沙箱对共享 gitdir 的限制。缓存 fetch、ref 更新和 worktree admin 修改有互斥，任务工作目录有独立分支。

`local_directory` 有两种语义：

- direct：Agent 直接修改用户目录，摩擦低但任务间不隔离；
- worktree：在 daemon 管理目录为每个任务建立 worktree，可并行开发，结束时 finalize/清理。

因此，同一项目两个任务并行是支持的，但“Git 工作副本隔离”不等于“依赖、进程、网络、凭据隔离”。如果任务会运行不可信脚本，仍需容器或 VM。

## 7. 前端架构

### 7.1 共享包

- `packages/core`：API client、schema、React Query、认证、realtime、hooks 和客户端状态；
- `packages/ui`：与业务数据解耦的视觉组件；
- `packages/views`：Issue、Project、Agent、Chat、Autopilot、Runtime、Skill、Plugin 等业务页面；
- `plugin-sdk`：插件声明和前端扩展契约。

状态约定是 React Query 管理服务端状态，Zustand 管理客户端状态；`views → core + ui`，而 `core/ui` 不依赖 `views`。共享 views 不应直接依赖 Next Router，使 Web 和 Desktop 能复用业务页面。

### 7.2 Web、Desktop、Mobile

Web 是较薄的 Next.js Shell。Desktop 的 Electron main/preload/renderer 形成明确权限边界，main 负责 daemon/CLI、更新、文件下载、原生菜单和 IPC，renderer 复用共享 views。Preload 暴露的 daemon API 可见于 [apps/desktop/src/preload/index.ts](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/apps/desktop/src/preload/index.ts)。

Mobile 不是 Web 的简单壳，而是 Expo Router 客户端，保有自己的 API/query/state/realtime/UI 层。这样能针对移动体验优化，但也扩大了契约重复和功能漂移风险；当前移动端功能面明显窄于 Web/Desktop。

## 8. 集成系统

### 8.1 Channel

Slack、Lark/Feishu、DingTalk、WeCom、Telegram 共享 Channel Engine 抽象：入站消息转成内部 Chat/Issue/Task，出站统一处理回复、线程、媒体、typing indicator 和投递账本。各渠道只实现平台特有的签名、身份、格式和发送 API。

### 8.2 VCS

GitHub 有单独的 GitHub App、Webhook、Check Suite 和 PR 语义；GitLab/Gitea/Forgejo 走 token-based Provider 抽象，入口可见 [server/internal/integrations/vcs/vcs.go](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/integrations/vcs/vcs.go)。这意味着“能 `git clone`”只解决代码获取，并不能自动获得 MR 列表、diff、评论、审批、状态回写和 webhook。

对 CodeHub 的建议路径：

1. 先验证它是否完整兼容某个现有 Provider 的 REST API、Webhook 和鉴权，而不是只看页面长得像 GitLab；
2. 若兼容，在 Provider base URL、TLS/CA、Token 和 clone URL 层配置即可；
3. 若只兼容原生 Git，使用 `local_directory + worktree` 或任务启动脚本绕过内置 checkout，但 MR 自动化仍不可用；
4. 若 API 有差异，新建 `CodeHubProvider`，实现仓库/MR/diff/comment/status/webhook 映射，并加契约测试；不要把 CodeHub 条件散落到 TaskService 和 daemon。

### 8.3 Plugin、MCP 与 Skills

Plugin 是持久化扩展系统，不只是 UI 插件。它包含冻结的版本 manifest/files、安装、授权、配置、加密 Secret、签名 Hook、短期 callback Token、Action API、MCP tool 审批、Surface 与 Skill 贡献。公网 Hook 有 SSRF 防护并拒绝 redirect。

Skills 有三种来源：服务端内置元技能、Workspace 持久 Skill/Skill File、daemon 本地导入；Agent 通过绑定获得 Skill。对通用 Code Review/编码 Skill，推荐单独的版本化共享仓库作为权威源，按版本发布，由多个 Multica Project/Agent 引用，不要复制到每个项目再独立漂移。

### 8.4 Autopilot

Autopilot 把版本化 Runbook、Assignee、Trigger、Quota、Subscriber/Collaborator 与 Run 分开。Schedule/Webhook/API/Manual 统一进入 dispatch；Webhook 先持久化 delivery 和幂等 Run，再由租约 Worker 执行。`create_issue` 在 Runtime 离线时仍可创建可审计任务；`run_only` 则会在离线时记录 skipped。

## 9. 安全模型

认证与授权不是单一 JWT，而是多层凭据：

| 凭据 | 主要用途 |
|---|---|
| 浏览器 HttpOnly Session/JWT + CSRF | Web 用户请求 |
| `mul_` PAT / Cloud Node PAT | CLI、兼容接口与机器访问 |
| daemon token | daemon 控制 API 与机器身份 |
| `mat_` task token | Agent 仅访问本任务允许的操作 |
| plugin callback/action token | 插件回调与 Action API |

Workspace membership、Agent invocation access、scope authorizer 和来源对象校验叠加在凭据之上。插件 Secret 使用加密存储，Hook 使用签名，任务 Token 取消后撤销。

最重要的安全边界仍在 daemon 主机：任务进程继承 daemon 用户的 HOME/XDG 环境，以便使用 Git、GitHub CLI、AWS 或 Agent Provider 登录态。官方实现没有文件系统沙箱。生产上应为 daemon 使用专用低权限用户、最小权限 deploy key、受控网络出口和每任务容器/VM；不要把个人开发机的全量凭据暴露给无人值守 Agent。

另一个需要显式决策的点是 Redis 缺失时部分保护会降级，例如插件 rate limit 选择 fail-open。单机开发合理，公网生产需要强制 Redis 或外层网关限流。

## 10. 部署、可观测性与恢复

### 10.1 部署

本地开发有 Makefile/dev-env；自托管支持 Docker Compose；Kubernetes 通过 Helm 部署 frontend、backend 和 PostgreSQL，也可切换到外部 PostgreSQL。Backend entrypoint 在启动前执行迁移，并用 PostgreSQL advisory lock 避免多副本迁移竞态。探针区分不依赖数据库的 liveness 与依赖数据库的 readiness。

多节点 Realtime、请求存储和租约协调可使用 Redis；默认 Helm backend 仍是 1 副本、Recreate 策略，说明横向能力存在，但默认模板偏单节点。上传文件可挂 PVC，Chart 提供部分 PrometheusRule。

### 10.2 可观测性与交付门禁

Prometheus 指标覆盖业务事件、数据库连接池、Realtime、Channel Media 和 Seat Capacity；metrics 默认关闭，样例建议只绑定 loopback 或增加鉴权。结构化日志和 Analytics 覆盖任务失败、Claim 慢请求、使用量和 Run Trace。

CI 按前端、后端、SQLC、Desktop 和 Mobile 路径过滤：Node 22 构建/类型/lint、分片前端测试、Go 1.26 配 PostgreSQL 17/Redis 7、Helm/配置/entrypoint 检查、Desktop 打包烟测和 Mobile 验证。Tag 发布增加 race 测试、`govulncheck`、GoReleaser、多架构 backend/web 镜像、OCI Helm Chart 和 Desktop 产物。

### 10.3 灾备缺口

仓库内没有完整的生产数据库/对象存储自动备份、保留、完整性校验、恢复脚本、RPO/RTO 或恢复演练 Runbook。内置 PostgreSQL 是单副本 RWO PVC；PVC 不是备份。CI artifact retention 也不能替代生产备份。

符合可替换服务器原则的生产部署至少需要：

- 外部托管 PostgreSQL 或定时 off-host 备份，含 PITR/WAL、保留规则和恢复演练；
- S3 版本化/复制，或本地 uploads 的 off-host 快照与校验；
- 私有 Git 仓库保存 Helm values 模板、部署脚本和非 Secret 配置；
- Secret Manager 中的 JWT、数据库、VCS、Channel、Plugin 与对象存储凭据清单；
- 明确 RPO/RTO、DNS/TLS 恢复、迁移版本校验、健康检查与回滚步骤；
- 定期在干净环境恢复 Server、数据库、对象和 daemon 注册关系。

## 11. 架构优势

1. **控制与执行分离**：远端机器只需 daemon 出站连接，便于跨系统、跨网络部署。
2. **数据库事实源**：任务、Webhook 和 Autopilot 都强调持久化与幂等，断线恢复比纯内存 Agent Orchestrator 可靠。
3. **权限按角色拆分**：人、daemon、task、plugin 有不同凭据，Task Token 能显著降低横向越权面。
4. **Runtime 适配层独立**：新增 Agent CLI 不需要重写项目、Issue、Chat 和 Task 模型。
5. **Git 并行能力完整**：bare cache、worktree、分支、锁、复用和 GC 形成闭环，并有 Windows/Linux 专项测试。
6. **前端复用边界清楚**：Web/Desktop 共享业务 views，Electron 原生能力留在 main/preload。
7. **集成采用适配器/账本**：Channel、VCS、Plugin、Autopilot 的外部副作用大多有统一接口或持久投递记录。

## 12. 主要风险与优先级

| 优先级 | 风险 | 影响 | 建议 |
|---|---|---|---|
| P0 | 无生产级灾备闭环 | 单机/PVC 丢失可能永久丢数据 | 增加版本化 DR Runbook、自动备份、校验与恢复演练 |
| P0 | daemon 无硬沙箱且继承用户凭据 | Agent/依赖脚本可读取或外传主机可达资源 | 专用用户 + 容器/VM + 最小凭据 + 出口策略 |
| P1 | daemon、TaskService、daemon handler 等巨石文件 | 修改半径大、评审困难、回归风险集中 | 按 Claim、Lease、Environment、Session、Result 子域拆分并冻结接口 |
| P1 | CodeHub/企业 VCS 兼容易与 checkout 混淆 | clone 成功但 MR、评论、状态回写失败 | 建立 VCS Provider 契约测试与能力探测，不在上层散布平台分支 |
| P1 | 无数据库外键 | 应用 bug 可制造孤儿或跨 Workspace 引用 | 为关键引用增加周期一致性检查、清理 Job 和迁移验证 |
| P1 | Redis 可选且部分保护 fail-open | 多节点一致性、限流和实时能力会降级 | 生产 Profile 强制 Redis，并在 readiness 暴露降级状态 |
| P2 | Mobile 独立实现 API/state/realtime | 功能和协议容易与 Web/Desktop 漂移 | 将 schema/client 契约继续下沉到共享包，建立跨端契约测试 |
| P2 | 约 440 组迁移、领域表约百张 | 升级和回滚窗口越来越复杂 | 发布前执行真实数据量迁移演练、记录兼容窗口和不可逆迁移 |

## 13. 低摩擦改进建议

低摩擦不等于隐藏所有复杂度，而是让常见路径只需声明意图，把分叉和失败变成可见、可恢复的状态。建议按以下顺序改进：

1. 建立 **Capability Probe**：Runtime、VCS、Channel、Storage 在项目配置时就显示可用能力和缺口，避免到任务执行才失败。
2. 将“代码 Review”固化为模板：MR URL → Provider 拉取 → 创建 Issue/Chat → 绑定 Review Agent + Skill → Worktree → 生成结构化意见 → 人确认后回写。
3. 复审使用 MR 当前 head SHA 作为幂等键；作者 force-push 后，保存上次 reviewed SHA 与评论指纹，只重评新 head，并自动标记已解决/仍存在/新增问题。
4. Skill 使用独立共享仓库、语义版本和变更日志；Multica 项目只 pin 版本。编辑后自动跑 fixture、lint 和最小真实仓库烟测，再发布版本。
5. 把手工路由脚本变成版本化 Router Plugin/Service：输入 Provider、仓库、标签、文件路径、CODEOWNERS、风险分数，输出 Agent/Skill/优先级；保存决策原因和规则版本。
6. 对 CodeHub 先实现最小竖切：鉴权 → 一个仓库 → 一个 MR → diff → 草稿评论，不要同时铺开全部 Provider 能力。

## 14. 分析边界

- 已覆盖所有顶层交付单元、主要 Go/TS 模块、数据库领域、HTTP/Realtime/daemon 边界、Agent Runtime、Git 隔离、Channel/VCS/Plugin/Skill/Autopilot、安全、部署、CI、可观测性和灾备。
- 没有启动所有外部 Provider 的真实账号做端到端测试，因此第三方 API 行为以实现和测试契约为准。
- 没有进行生产负载测试、故障注入或安全渗透测试；性能和安全结论只陈述源码能证明的边界。
- 仓库开始前已有 `apps/desktop/build/*` 删除状态，本次分析未恢复或修改这些用户变更。

