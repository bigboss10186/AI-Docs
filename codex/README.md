# Codex

这个目录用于归档 OpenAI Codex 相关的源码阅读、架构理解、agent 机制和特性分析文档。当前重点是 `codex-rs/` Rust workspace 的分层、目录职责和主要调用链。

## 范围

- `architecture/`: 架构图、crate 分层、目录职责和调用链索引。
- `agent-loop/`: 用户输入、模型请求、工具调用和最终响应。
- `tool-execution/`: shell、apply_patch、MCP、动态工具、审批和沙箱。
- `context/`: 模型上下文、历史消息、context fragment、缓存和大小限制。
- `memory/`: agent 记忆系统、上下文压缩、长期 memory 和 thread 历史治理。
- `app-server/`: JSON-RPC API、thread 生命周期、桌面端交互。
- `tui/`: 终端 UI、事件循环、组件结构和 snapshot 测试。
- `plugins-skills/`: plugin、skill、MCP、extension 的边界和加载流程。

## 已归档笔记

- [Codex Rust Workspace 分层索引](architecture/codex-rs-index.md): 从源码阅读角度梳理 `codex-rs/` 的入口层、协议层、核心 agent 层、工具执行层、扩展层、状态配置层和支撑库。
- [Codex 记忆系统实现分析](memory/codex-memory-system.md): 分析短期会话历史、上下文注入、上下文压缩、replacement history 和长期 memories DB。

## 归档规则

1. README 只做入口索引，不写长篇源码分析。
2. 具体源码分析放到对应专题目录。
3. 一篇笔记优先围绕一个机制、一次调用链或一个核心问题。
