# Database Kernel Index

## 范围

数据库内核相关知识，包括存储、WAL/replay、事务、MVCC、空间回收、存储引擎差异和问题定位。

## 主题索引

- 存储层: page、tuple、heap、索引、空间回收。
- WAL/replay: WAL record、rmgr、redo、checkpoint、restartpoint。
- 事务与可见性: xid、snapshot、visibility、vacuum。
- 存储引擎: PostgreSQL heap，openGauss ASTORE/USTORE/CStore/D-Store。
- 问题定位: 现象、日志、目录结构、源码路径和机制解释。

## 已归档笔记

- [PostgreSQL](../postgres/README.md)
- [GaussDB/openGauss](../gaussdb/README.md)
- [数据库问题定位](../diagnostics/README.md)
- [存储层概念分层](../postgres/storage/storage-layer-concepts.md)
- [Page Prune、VACUUM、TRUNCATE 与空间回收](../postgres/storage/cleanup.md)
- [WAL 回放专题](../postgres/replay/wal-replay-study.md)
- [Replay 源码地图](../postgres/replay/source-map.md)
- [openGauss/GaussDB 存储引擎分析](../gaussdb/storage-engines/storage-engines.md)
- [表空间回放软链接冲突分析](../diagnostics/gaussdb/tablespace-symlink-replay.md)

## 待整理条目

- MVCC、snapshot 和 visibility。
- Buffer manager 与 page 生命周期。
- Checkpoint/restartpoint 与 replay 边界。
- PostgreSQL 与 openGauss 在 WAL/replay 上的关键差异。
