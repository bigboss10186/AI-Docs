# Personal Knowledge Base

这个目录是个人知识库入口，用来沉淀数据库、AI/Codex、算法和问题定位相关的知识，让知识可以被归档、检索、串联和复用。

## 主题入口

- [主题索引](indexes/README.md): 跨主题索引、概念关系和知识归档入口。
- [PostgreSQL](postgres/README.md): PostgreSQL 内核机制、源码坐标和专题笔记。
- [GaussDB/openGauss](gaussdb/README.md): GaussDB/openGauss 机制差异、存储引擎和源码分析。
- [Codex](codex/README.md): OpenAI Codex 源码结构、agent 机制和工具执行相关笔记。
- [算法](algorithms/README.md): 算法、数据结构和复杂度优化模式。
- [问题定位](diagnostics/README.md): 数据库问题定位、故障分析和案例复盘。
- [临时收集](inbox/README.md): 尚未整理的想法、问题、链接和片段。
- [笔记模板](templates/README.md): 概念、源码阅读、排障复盘等模板。
- [Skills](skills/README.md): 面向 Codex 新会话的知识库说明和可复用工作流。

## 归档原则

1. 一级目录按知识域组织，例如数据库、AI/Codex、算法、问题定位。
2. 每个一级目录的 `README.md` 作为该知识域的索引，负责列出已有笔记和归档规则。
3. 新笔记优先写清楚结论、背景、证据、源码坐标和关联笔记。
4. 临时内容先放到 `inbox/`，整理后迁入对应知识域。
5. 跨主题内容放到 `indexes/` 做索引，不强行搬动原文。
6. 具体专题文件使用语义化文件名，避免继续堆泛泛的 `README.md`。

## 笔记元信息

建议在笔记头部使用轻量元信息:

```md
---
topic: postgres
type: source-map
status: draft
created: 2026-07-06
updated: 2026-07-06
---
```

状态约定:

- `draft`: 草稿，还在补充。
- `usable`: 信息完整，已经能帮助复查。
- `solid`: 比较成熟，可以长期引用。

类型约定:

- `concept`: 概念解释。
- `source-map`: 源码坐标和调用关系。
- `debug`: 排障记录。
- `study-note`: 普通笔记。
- `summary`: 阶段性整理。
- `question`: 问题清单。

## 数据库内核笔记归档规则

当前数据库笔记按“共通概念”和“项目特有概念”分层:

1. PostgreSQL 和 openGauss/GaussDB 都适用的基础概念，优先放到 `postgres/`。例如 page、tuple、heap、btree、WAL/XLOG、VACUUM、rmgr、smgr、checkpoint、redo。
2. openGauss/GaussDB 特有或明显扩展的内容，放到 `gaussdb/`。例如 ASTORE、USTORE、CStore、D-Store、uheap、ubtree、undo worker、GaussDB 部署和回放差异。
3. 如果一个主题既有共通概念又有 openGauss 特化实现，先在 `postgres/` 写通用模型，再在 `gaussdb/` 写差异和源码坐标，并互相链接。
