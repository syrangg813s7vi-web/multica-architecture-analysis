---
layout: default
title: Multica 如何通过 Runtime 连接本地与远程 Code Agent
permalink: /runtime-code-agent/
---

# Multica 如何通过 Runtime 连接本地与远程 Code Agent

把任务分配给一个 Agent，为什么代码会在另一台机器上开始运行？公司自研的 Code Agent，又应该接到 Multica 的哪一层？

理解这两个问题，需要先拆开三个概念：**Agent 是业务配置，Runtime 是可用执行端的记录，Backend 是具体的协议适配代码。** 真正组织执行的是运行在执行机器上的 daemon。

> 本文为独立源码分析，发表于 2026-09-16，基于固定版本 [`c1a61e1e8`](https://github.com/multica-ai/multica/tree/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd)。不代表最新版本的全部行为，也不表示已经完成自研 Agent 的接入测试。

## 先看架构：谁负责什么？

[![Runtime 与 Code Agent 架构图]({{ '/assets/previews/runtime-code-agent.png' | relative_url }})]({{ '/diagrams/runtime-code-agent.html' | relative_url }})

[全屏打开交互式架构图 →]({{ '/diagrams/runtime-code-agent.html' | relative_url }}) 图中节点提供固定版本的源码引用。

| 概念 | 职责 | 不应该混淆成什么 |
| --- | --- | --- |
| Multica Agent | 保存角色、指令、模型选择、Runtime 绑定等业务配置 | 不等于一个常驻 CLI 进程 |
| Runtime | 表示平台登记的执行能力，关联 daemon 和协议类型 | 不等于模型 API，也不是独立执行服务 |
| daemon | 注册执行能力、领取任务、准备工作环境、管理执行并回传结果 | 不只是远程终端代理 |
| Backend | 将统一请求转换为某一种 Code Agent 的启动方式和通信协议 | 不只是一个命令路径 |
| Code Agent CLI | 运行 Agent 循环，调用模型和工具，读取或修改代码 | 不由 Multica 服务端直接替代 |

架构图中的框是职责划分，不意味着每个框都需要独立部署。Backend 在 daemon 的程序中工作；Code Agent 通常作为执行机器上的子进程启动。

## 远程连接：不是平台每次 SSH 登录机器

在这条源码执行链中，远程机器上的 daemon **主动连接 Multica 服务端**。你在浏览器操作平台，而不是让浏览器直接连接远程 CLI。

两条连接应分开理解：

1. **跨机器控制连接：远程 daemon → Multica 服务端。** daemon 带认证信息建立 WebSocket 控制连接，端点是 `/api/daemon/ws`。服务端验证身份、Runtime 访问权限及相关工作区范围。
2. **执行机器内的进程通信：daemon 中的 Backend → Code Agent。** Backend 启动本机 CLI，并通过它支持的协议传入任务、接收事件。

服务器通知任务可用，不等于已经完成任务领取。当前实现优先使用 WebSocket RPC `tasks.claim` 领取任务，并支持 HTTP 回退。对于“请求可能已经发出，但响应丢失”的情况，代码会等待安全窗口，而不是立刻重复领取。

因此，SSH 可以用于初次安装和日常运维，但不属于这里的正常任务派发链路。远程机器也不需要再部署一整套 Multica Web/API 服务；它需要 Multica CLI／daemon、相应的 Code Agent CLI，以及项目所需的工具和权限。

来源：[客户端连接与认证](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/wakeup.go#L107)、[服务端访问校验](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/handler/daemon_ws.go#L11)、[WS 优先的任务领取](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/wsrpc.go#L308)。

## 一次任务如何执行？

[![任务执行流程图]({{ '/assets/previews/runtime-execution.png' | relative_url }})]({{ '/diagrams/runtime-execution.html' | relative_url }})

[全屏打开执行流程图 →]({{ '/diagrams/runtime-execution.html' | relative_url }})

### 1. 发现并注册执行能力

daemon 发现本机可用的 Agent CLI，也会读取工作区的自定义 Runtime Profile。对于自定义 Profile，它解析命令路径、检查协议类型，再登记对应的 Runtime。命令不可用时，不能仅凭平台上的一条配置就获得执行能力。

### 2. 绑定 Agent，创建任务

在 Multica 中选择 Agent 使用的 Runtime。任务进入平台调度流程，再由对应执行端领取。**模型选择和执行机器选择是两件事**：前者决定执行请求里的模型参数，后者决定在哪里、通过哪种 CLI 执行。

### 3. 准备目录、上下文和配置

daemon 准备任务工作目录、指令、Skills、MCP 配置和环境变量。不同 Code Agent 的原生读取约定不同，因此不能假设所有指令都通过一个 `SystemPrompt` 字段传递。

源码明确指出，多数适配器的任务说明通过工作目录里的 `AGENTS.md`、`CLAUDE.md` 等上下文文件交付；只有部分执行端需要内联系统指令。接入新 Backend 时，忽略这点可能导致“能启动，却没有收到平台指令”。

### 4. 分别决定协议和可执行文件

daemon 构造 Backend 时，传入两类关键配置：

- `provider / protocol_family`：决定采用哪个协议适配器。
- `ExecutablePath`：决定实际启动哪个程序；`LaunchPrefix` 承载固定启动参数。

核心调用是 `agent.ResolveBackend(provider, config)`，随后调用统一接口：

```go
Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error)
```

`ExecOptions` 包括工作目录、模型、超时、续接会话 ID、自定义参数等。当前模型解析先考虑 Agent 显式选择，其次是 daemon 的提供方默认配置；仍为空时交给对应 CLI 使用自己的默认值。

来源：[统一接口及上下文约定](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent/agent.go#L18)、[Backend 构造与模型解析](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/daemon.go#L7480)。

### 5. 启动 CLI，归一化过程事件

Backend 隐藏各家 CLI 的差异。例如：

| 执行端 | 当前源码使用的通信方式 |
| --- | --- |
| Claude Code | CLI 的 `stream-json` 输入与输出 |
| Codex | 启动 `codex app-server --listen stdio://`，使用 app-server 协议 |

适配器把原生输出转换为统一的 `Session.Messages` 与 `Session.Result`。消息可包含文本、思考、工具调用、工具结果、状态和错误；结果承载最终结论。

daemon 在执行期间消费并上报消息，结束后报告最终状态。流程图下排表示回流方向，**不是等整个任务结束才开始显示日志**。实际实现另有取消、超时与会话恢复失败处理，本图没有展开所有异常分支。

来源：[Claude 协议参数](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent/claude.go#L716)、[Codex 启动方式](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent/codex.go#L257)、[统一消息通道](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent/agent.go#L137)、[消息上报](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/daemon.go#L8279)。

## 自研 Code Agent 应接在哪一层？

### 已兼容某种现有协议：优先使用 Runtime Profile

如果公司命令只是封装已有 CLI，或确实实现了它所需的通信协议，可以利用自定义 Profile，把协议类型和公司命令关联起来。例如下面仅适用于**兼容 Codex app-server 协议**的命令：

```sh
multica runtime profile create \
  --protocol-family codex \
  --command-name company-codeagent \
  --display-name "Company Code Agent"
```

daemon 在执行机器上解析 `company-codeagent`。必要时，可以通过 `multica runtime profile set-path` 为某台机器指定本地可执行路径；这类路径映射不发送到服务端。

**把 `protocol_family` 填为 `codex` 不会自动转换协议。** 任意“输入一段话、输出一段文本”的程序，并不因此兼容 Codex Backend。自定义参数也不是所有 Backend 都统一支持，应以适配器实际消费的字段为准。

来源：[Runtime Profile 设计与命令](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/cmd/multica/cmd_runtime_profile.go#L20)、[daemon Profile 注册](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/internal/daemon/daemon.go#L2752)。

### 协议独立：增加转换器或 Backend

如果内部 CLI 使用独立协议，可以在它外面增加协议转换程序，也可以在 Multica 中实现新的 Backend。后者还需检查支持类型白名单、发现与注册、配置校验和测试等路径，不能只增加一个启动命令。

建议用以下能力作为接入验收清单：

- 无交互启动，正确接收任务、目录和模型配置。
- 文本、工具调用、错误能转成平台消息，完成状态不会漏报。
- 取消和超时能正确终止执行，不遗留失控进程。
- 续接 ID 有明确语义；不支持续接时显式说明，不伪装成功。
- 指令、Skills 和 MCP 配置能够被内部 Agent 真正读取。

来源：[Backend 工厂与支持类型](https://github.com/multica-ai/multica/blob/c1a61e1e863eb62ddd7b5fd5ab5ff85391f212fd/server/pkg/agent/agent.go#L300)。以上清单是基于接口职责提出的工程建议，不是已经完成的兼容性认证。

## 远程部署最容易忽略的三个条件

**服务端地址可达。** 执行机器需要访问 Multica 服务端，代理链路需支持 WebSocket。执行机器上的回环地址指向它自己，不会自动指向操作浏览器的电脑。

**执行账号具备必要权限。** 代码仓访问、CLI 登录和模型认证都要在 daemon 所使用的系统账号下成立；“另一个账号手动执行成功”不能证明 daemon 可以执行。

**工作目录隔离不等于权限隔离。** Git worktree 可以分开代码工作目录，但不会自动限制 Agent 对宿主机文件和凭据的访问。建议使用专用低权限账号，并明确可访问的仓库和凭据范围。这是部署建议，不是 Runtime 自带的安全保证。

## 结论

Multica 对接远程 Agent 的关键，不是远程 shell，而是**平台与 daemon 的控制协议，以及 daemon 与 Code Agent 的执行协议**。

Runtime 回答“哪个执行端可用”，daemon 回答“如何准备并管理任务”，Backend 回答“如何与这家 Code Agent 通信”。把这三层拆开，就能判断一个内部 Agent 的接入究竟只是配置工作，还是需要协议适配开发。

[返回架构分析首页]({{ '/' | relative_url }}) · [阅读全仓库分析]({{ '/analysis/' | relative_url }})
