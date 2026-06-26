# GaussDB / openGauss Docs

这个目录用于沉淀 GaussDB/openGauss 相关源码分析、分布式部署问题和故障排查笔记。

## 专题

- [表空间回放软链接冲突分析](tablespace-symlink-replay/README.md): 分析 openGauss 在回放 `CREATE TABLESPACE` 时，为什么数据目录会带节点名隔离，但 `pg_tblspc/<tablespace_oid>` 软链接入口仍可能发生冲突。
