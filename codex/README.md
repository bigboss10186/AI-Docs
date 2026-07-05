# Codex Docs

这个目录用于沉淀 OpenAI Codex 相关的源码阅读、架构理解和特性分析文档。当前重点是帮助理解 Codex 代码仓中 `codex-rs/` Rust workspace 的分层、目录职责和主要调用链。

## 当前专题

- [Codex Rust Workspace 分层索引](architecture/codex-rs-index.md): 从源码阅读角度梳理 `codex-rs/` 的入口层、协议层、核心 agent 层、工具执行层、扩展层、状态配置层和支撑库。

## 目录规划

后续文档可以按下面方式组织：

1. `architecture/`: 架构图、crate 分层、目录职责和调用链索引。
2. `agent-loop/`: 一次用户输入如何进入模型请求、工具调用和最终响应。
3. `tool-execution/`: shell、apply_patch、MCP、动态工具、审批和沙箱。
4. `context/`: 模型上下文、历史消息、context fragment、缓存和大小限制。
5. `app-server/`: JSON-RPC API、thread 生命周期、桌面端交互。
6. `tui/`: 终端 UI、事件循环、组件结构和 snapshot 测试。
7. `plugins-skills/`: plugin、skill、MCP、extension 的边界和加载流程。

## 阅读方式

这个目录下的 README 只作为文档集合入口。具体源码分析应放到对应专题目录中，并在这里添加索引链接。
