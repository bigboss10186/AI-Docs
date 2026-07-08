# GaussDB / openGauss

这个目录用于归档 GaussDB/openGauss 相关源码分析、分布式部署、内核机制和 PostgreSQL 差异化内容。

## 范围

- 存储引擎: ASTORE、USTORE、CStore、D-Store。
- openGauss 特有机制: uheap、ubtree、undo worker、节点目录隔离等。
- 与 PostgreSQL 的差异: 数据组织、更新模型、MVCC/undo、WAL/replay 行为。

## 已归档笔记

- [openGauss/GaussDB 存储引擎分析](storage-engines/storage-engines.md): 对比 ASTORE、USTORE、CStore、D-Store 的数据组织、更新模型、MVCC/undo、列存 CU、page/TD 结构和适用场景。

## 相关案例

问题定位类文档统一放到同层的 `diagnostics/gaussdb/` 下:

- [表空间回放软链接冲突分析](../diagnostics/gaussdb/tablespace-symlink-replay.md): 分析 openGauss 在回放 `CREATE TABLESPACE` 时，为什么数据目录会带节点名隔离，但 `pg_tblspc/<tablespace_oid>` 软链接入口仍可能发生冲突。

## 归档规则

1. PostgreSQL 和 openGauss 共通的基础概念，优先放到 `../postgres/`。
2. openGauss/GaussDB 特有机制、产品行为和差异分析，放到当前目录。
3. 具体故障和排查过程，放到 `../diagnostics/gaussdb/`，再从这里建立链接。
