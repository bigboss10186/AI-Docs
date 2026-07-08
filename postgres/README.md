# PostgreSQL

这个目录用于归档 PostgreSQL 内核机制、源码坐标和专题笔记。

## 范围

- `storage/`: Page、tuple、heap/uheap、btree/ubtree、WAL/XLOG、VACUUM 等存储层专题。
- `replay/`: WAL redo/replay、checkpoint、restartpoint、WAL insert 指针推进等专题。

## 已归档笔记

- [存储层概念分层](storage/storage-layer-concepts.md)
- [Page Prune、VACUUM、TRUNCATE 与空间回收](storage/cleanup.md)
- [WAL 回放专题入口](replay/wal-replay-study.md)
- [Replay 源码地图](replay/source-map.md)
- [Replay 总览流程](replay/overview-flow.md)

## 笔记要求

每个专题尽量保留以下信息:

1. 结论: 先写这个机制的核心判断。
2. 概念: 解释关键术语和边界。
3. 交互: 描述多个机制之间如何互相影响。
4. 源码坐标: 定位到文件、函数和关键结构。
5. 关联笔记: 链接到相关概念或案例。
