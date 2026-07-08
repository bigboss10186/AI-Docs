# WAL Replay Notes

这个专题用于理解 PostgreSQL 的 WAL 回放整体概念，以及 checkpoint、restartpoint、WAL insert 指针推进之间的关系。

## 文档索引

1. [主备 WAL 全链路流程图](overview-flow.md)
2. [Checkpoint 基本原理](concepts/checkpoint.md)
3. [smgr 与 rmgr](concepts/smgr-rmgr.md)
4. [PostgreSQL 回放方式特性文档](features/replay-modes.md)
5. [WAL Record 如何作用到 Page](features/page-redo-flow.md)
6. [WAL insert head/tail 与写入推进](concepts/insert-pointers.md)
7. [源码地图](source-map.md)

## 核心结论

- WAL replay 的起点不是“最新 checkpoint 记录本身”，而是 checkpoint 记录中保存的 `redo` 指针。这个指针表示崩溃恢复必须从哪里重新应用 WAL。
- 普通在线 checkpoint 会先写一个 `XLOG_CHECKPOINT_REDO` 记录来确定新的 redo 点，然后刷脏页，最后写真正的 `XLOG_CHECKPOINT_ONLINE` checkpoint 记录。
- shutdown checkpoint 没有并发 WAL 插入，可以让 checkpoint 记录自身承担 redo 点的角色。
- recovery/standby 中不能创建普通 checkpoint，只能在已经回放到安全 checkpoint 记录后创建 restartpoint。
- WAL insert 端的“head”主要对应 `Insert->CurrBytePos`，即已经预留出去的 WAL 末端；写盘/flush 端通过 `LogwrtRqst`、`LogwrtResult` 和 `WaitXLogInsertionsToFinish()` 确保只写已经完成拷贝的 WAL。
- 主备链路可以先按“主机 WAL insert/flush -> walsender 发送 -> walreceiver 接收/flush -> startup process replay -> restartpoint 推进”这条主线理解。
- 按 redo 执行模型看，PostgreSQL 社区版核心是 startup process 串行回放；`recovery_prefetch` 是 I/O 预读优化，不是并行 redo。
- `rmgr` 是 WAL record 的资源管理器分发层；`smgr` 是 relation 物理文件和 block I/O 的 storage manager 抽象，`RM_SMGR_ID` 是二者在 WAL redo 体系中的连接点。
- 页面级 redo 的核心链路是：解析 WAL record -> 按 RmgrId 分发 -> 定位 relation/fork/block -> 处理 FPI 或 page LSN 判断 -> 修改 page -> 设置 LSN、标脏、释放 buffer。

## 外部背景资料

- PostgreSQL 官方文档：[Write-Ahead Logging (WAL)](https://www.postgresql.org/docs/current/wal-intro.html)
- PostgreSQL 官方文档：[WAL Configuration](https://www.postgresql.org/docs/17/wal-configuration.html)
- PostgreSQL 官方文档：[WAL Internals](https://www.postgresql.org/docs/current/wal-internals.html)
