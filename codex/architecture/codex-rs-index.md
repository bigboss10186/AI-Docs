# Codex Rust Workspace Index

这个目录用于沉淀 OpenAI Codex 代码仓的阅读笔记，当前重点是 `codex-rs/` 这个 Rust workspace。它可以作为后续深入阅读源码时的索引：先知道有哪些层、每一层负责什么，再按问题去定位对应 crate 和目录。

## 总体分层

`codex-rs/` 可以粗略理解为一个多入口的 agent 运行时。外层负责 CLI、TUI、桌面端/app-server 等交互入口，中间层负责协议、会话、上下文和 agent 编排，底层负责模型请求、工具调用、命令执行、沙箱、文件系统、状态存储和扩展能力。

```text
用户入口层
  cli / tui / app-server / app-server-daemon
        |
协议与 API 层
  protocol / app-server-protocol / exec-server-protocol / code-mode-protocol
        |
核心 agent 编排层
  core / core-api / context-fragments / prompts / message-history
        |
模型与后端访问层
  codex-client / backend-client / codex-api / model-provider / chatgpt / ollama / lmstudio
        |
工具执行与沙箱层
  exec / exec-server / sandboxing / execpolicy / linux-sandbox / apply-patch / shell-command
        |
扩展与集成层
  codex-mcp / mcp-server / plugin / skills / core-skills / connectors / ext/*
        |
状态、配置和支撑库
  config / state / thread-store / codex-home / git-utils / file-system / utils/*
```

这个分层不是严格的编译依赖图，而是阅读源码时更容易建立心智模型的功能分层。

## 关键目录

- `codex-rs/Cargo.toml`: Rust workspace 入口，列出所有 crate 和 workspace 级依赖。理解整个 Rust 部分时先看这里。
- `codex-rs/cli/`: 命令行入口，处理 `codex` 命令、登录、doctor、插件、MCP、沙箱调试等子命令。
- `codex-rs/tui/`: 终端交互界面。负责聊天 UI、输入框、审批弹层、历史展示、线程切换、状态栏和 app-server 事件接入。
- `codex-rs/app-server/`: 桌面端和外部客户端使用的本地服务层，负责 JSON-RPC 请求处理、线程生命周期、配置管理、文件/搜索/Git/MCP/插件等能力编排。
- `codex-rs/app-server-protocol/`: app-server 的协议类型定义，Rust 类型和生成的 TypeScript schema 都从这里维护。
- `codex-rs/core/`: 核心 agent 运行逻辑。包含会话、上下文管理、工具调用、模型交互、审批、沙箱接入、状态和事件流等主要逻辑。
- `codex-rs/core-api/`: 对核心能力的 API 抽象，供其他 crate 以较小接口接入核心。
- `codex-rs/protocol/`: Codex 内部事件、请求、响应等基础协议类型。
- `codex-rs/codex-client/`, `backend-client/`, `codex-api/`: 和 OpenAI/Codex 后端交互的客户端与 API 类型。
- `codex-rs/model-provider/`, `model-provider-info/`, `models-manager/`: 模型提供方、模型元信息和模型列表管理。
- `codex-rs/chatgpt/`, `ollama/`, `lmstudio/`: 面向不同模型来源或服务形态的适配层。
- `codex-rs/exec/`: 本地命令执行相关能力，是工具调用落到系统命令时的重要路径。
- `codex-rs/exec-server/` 和 `exec-server-protocol/`: 远端或隔离执行服务及其协议。
- `codex-rs/execpolicy/` 和 `execpolicy-legacy/`: 命令执行策略、审批和安全规则相关逻辑。
- `codex-rs/sandboxing/`, `linux-sandbox/`, `windows-sandbox-rs/`, `bwrap/`: 跨平台沙箱和平台特定沙箱实现。
- `codex-rs/apply-patch/`: patch 应用工具，支撑 agent 修改文件时的结构化补丁流程。
- `codex-rs/codex-mcp/`, `mcp-server/`, `rmcp-client/`: MCP 客户端、服务端和连接管理相关代码。
- `codex-rs/plugin/`, `core-plugins/`, `skills/`, `core-skills/`, `ext/*`: 插件、skill 和扩展能力。`ext/` 下通常是按能力拆分的扩展包，例如 MCP、web search、image generation、memories、goal 等。
- `codex-rs/config/`: 配置结构、加载、编辑和 schema 生成相关逻辑。
- `codex-rs/state/`, `thread-store/`, `message-history/`: 本地状态、线程、消息历史、迁移脚本等持久化能力。
- `codex-rs/file-system/`, `file-search/`, `file-watcher/`: 文件系统抽象、搜索和监听。
- `codex-rs/git-utils/`: Git 相关通用能力。
- `codex-rs/prompts/`, `context-fragments/`, `response-debug-context/`: prompt、上下文片段和调试上下文。
- `codex-rs/otel/`, `analytics/`, `feedback/`: 可观测性、分析和反馈。
- `codex-rs/utils/*`: 小型通用工具 crate，例如路径、PTY、缓存、字符串、模板、插件工具、输出截断等。

## 核心层阅读重点

`core/` 是理解 Codex agent 的主要入口，但它比较大，建议按功能块读：

- `core/src/session/`: 会话生命周期、事件流和一次用户 turn 的运行边界。
- `core/src/agent/`: agent 编排逻辑，包括内置行为、控制流和模型驱动的执行过程。
- `core/src/context/` 与 `core/src/context_manager/`: 模型可见上下文如何组织、裁剪和注入。
- `core/src/tools/`: agent 可调用工具的定义、注册、调度和结果处理。
- `core/src/mcp_tool_call/`: MCP 工具调用路径。
- `core/src/unified_exec/`: 统一命令执行路径，连接工具调用、审批、沙箱和输出处理。
- `core/src/config/`: 核心层使用的配置视图。
- `core/src/sandboxing/`: 核心层如何选择和调用沙箱。
- `core/src/state/`: 核心运行时状态。

## 典型调用链

### CLI/TUI 本地对话

1. 用户从 `cli/` 启动命令，或进入 `tui/` 终端界面。
2. UI 层收集用户输入、当前目录、权限模式、模型配置等信息。
3. 请求进入 `core/`，创建或恢复 session。
4. `core/` 组装上下文，通过 `codex-client/`、`backend-client/` 或模型 provider 发起模型请求。
5. 模型返回消息或工具调用。
6. 工具调用进入 `core/src/tools/`，命令类工具进一步进入 `unified_exec`、`exec`、`execpolicy` 和 `sandboxing`。
7. 执行结果回到 `core/`，再作为 tool output 继续送回模型，直到本轮完成。
8. UI 层消费事件流，展示回答、diff、命令输出、审批请求和状态变化。

### App/Desktop 路径

1. 桌面端或外部客户端连接 `app-server/`。
2. `app-server-protocol/` 定义 JSON-RPC 方法、参数和返回类型。
3. `app-server/src/request_processors/` 按资源处理请求，例如 thread、config、git、fs、mcp、plugin 等。
4. 涉及 agent 对话时，app-server 创建或驱动 core session。
5. 事件通过 app-server 的 outgoing message/notification 流回到客户端。

### 工具执行路径

1. 模型提出工具调用。
2. `core` 根据工具名找到对应 handler。
3. 命令执行类工具进入统一执行层，先判断策略和审批要求。
4. 根据平台和配置选择沙箱或执行服务。
5. 执行输出被截断、结构化、记录，并返回给 agent。

## 阅读路线

第一次读建议按下面顺序：

1. `codex-rs/Cargo.toml`: 先建立 workspace 全局地图。
2. `codex-rs/cli/src/main.rs` 和 `cli/src/lib.rs`: 看命令行入口如何进入系统。
3. `codex-rs/tui/src/app.rs`、`tui/src/chatwidget.rs`: 看终端 UI 如何组织事件和输入输出。
4. `codex-rs/app-server/src/request_processors/`: 看桌面端/API 请求如何被分发。
5. `codex-rs/core/src/session/`: 看一次对话 turn 的生命周期。
6. `codex-rs/core/src/agent/`: 看 agent 如何驱动模型、工具和控制流。
7. `codex-rs/core/src/tools/` 与 `core/src/unified_exec/`: 看工具调用和命令执行。
8. `codex-rs/execpolicy/`、`sandboxing/`、`exec/`: 看执行安全边界。
9. `codex-rs/app-server-protocol/`、`protocol/`: 回头整理事件和 API 类型。

## 后续可扩展专题

后续可以在本目录继续拆专题：

1. `architecture/`: Codex Rust workspace 架构图、crate 依赖和调用链。
2. `agent-loop/`: 一次用户输入如何变成模型请求、工具调用和最终响应。
3. `tool-execution/`: shell、apply_patch、MCP、动态工具和审批策略。
4. `context/`: 模型上下文、历史、context fragment、缓存命中和大小限制。
5. `app-server/`: JSON-RPC API、thread 生命周期、桌面端交互。
6. `tui/`: ratatui UI 结构、事件循环、snapshot 测试。
7. `sandbox/`: macOS/Linux/Windows 沙箱与执行策略。
8. `plugins-skills/`: plugin、skill、MCP、extension 的边界和加载流程。

## 快速定位问题

- 想看命令从哪里进来：先看 `cli/`。
- 想看终端界面怎么画：先看 `tui/`。
- 想看桌面端/API 怎么调 Codex：先看 `app-server/` 和 `app-server-protocol/`。
- 想看 agent 如何运行：先看 `core/src/session/` 和 `core/src/agent/`。
- 想看模型请求：先看 `codex-client/`、`backend-client/`、`model-provider/`。
- 想看 shell 命令和审批：先看 `core/src/unified_exec/`、`exec/`、`execpolicy/`。
- 想看沙箱：先看 `sandboxing/`，再看平台特定目录。
- 想看 MCP：先看 `codex-mcp/`、`mcp-server/` 和 `core/src/mcp_tool_call/`。
- 想看插件和 skill：先看 `plugin/`、`skills/`、`core-skills/`、`ext/*`。
- 想看配置：先看 `config/` 和 `core/src/config/`。
- 想看本地持久化：先看 `state/`、`thread-store/`、`message-history/`。
