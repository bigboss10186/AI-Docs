# GaussDB / openGauss Docs

这个目录用于沉淀 GaussDB/openGauss 相关源码分析、分布式部署和内核机制笔记。

## 专题

- [openGauss/GaussDB 存储引擎分析](storage-engines/storage-engines.md): 对比 ASTORE、USTORE、CStore、D-Store 的数据组织、更新模型、MVCC/undo、列存 CU、page/TD 结构和适用场景。

问题定位类文档统一放到同层的 `diagnostics/gaussdb/` 下：

- [表空间回放软链接冲突分析](../diagnostics/gaussdb/tablespace-symlink-replay.md): 分析 openGauss 在回放 `CREATE TABLESPACE` 时，为什么数据目录会带节点名隔离，但 `pg_tblspc/<tablespace_oid>` 软链接入口仍可能发生冲突。
