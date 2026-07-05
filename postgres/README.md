# Postgres Study Docs

这个目录用于沉淀当前 PostgreSQL 代码仓的学习文档和可复用 skill，是 `docs` AI 文档库中的 PostgreSQL 专区。

## 目录结构

- `storage/`: Page、tuple、heap/uheap、btree/ubtree、WAL/XLOG、VACUUM 等存储层专题。
- `replay/`: WAL redo/replay、checkpoint、restartpoint、WAL insert 指针推进等专题。
- `skills/`: 后续可以放面向 Codex/Agent 的工作流、阅读模板、调试手册。

## 阅读方式

每个专题尽量包含三层内容：

1. 概念总览：先解释机制和关键术语。
2. 交互细节：描述多个机制之间如何互相影响。
3. 源码坐标：定位到当前仓库里的文件、函数和关键代码段。

当前第一版专题：

- [存储层概念分层](storage/storage-layer-concepts.md)
- [Page Prune、VACUUM、TRUNCATE 与空间回收](storage/cleanup.md)
- [WAL 回放专题入口](replay/wal-replay-study.md)
