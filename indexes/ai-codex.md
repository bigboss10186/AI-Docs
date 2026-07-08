# AI / Codex Index

## 范围

Codex 和 AI coding agent 相关知识，包括源码结构、agent loop、工具执行、上下文管理、审批、沙箱、插件和 skill。

## 主题索引

- 代码仓结构: workspace、crate、入口、协议层和核心 agent。
- Agent 机制: 用户消息、模型请求、工具调用、响应聚合。
- 工具执行: shell、apply_patch、MCP、审批、沙箱。
- 上下文管理: 历史消息、context fragment、压缩和大小限制。
- 记忆系统: 短期会话历史、上下文压缩、replacement history、长期 memory 抽取和 consolidation。
- App/TUI: 桌面端、JSON-RPC、终端 UI、事件和状态同步。

## 已归档笔记

- [Codex](../codex/README.md)
- [Codex Rust Workspace 分层索引](../codex/architecture/codex-rs-index.md)
- [Codex 记忆系统实现分析](../codex/memory/codex-memory-system.md)

## 待整理条目

- 一次用户请求在 Codex 中的完整调用链。
- 工具调用结果如何写回模型上下文。
- 审批和沙箱在代码层面的边界。
- plugin、skill、MCP、动态工具之间的职责差异。
- 长期 memory 的 stage-1 extraction 和 phase-2 consolidation 完整调度链路。
