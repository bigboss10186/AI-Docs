# WAL Replay Source Map

## 主路径

| 主题 | 文件 | 函数/结构 | 说明 |
| --- | --- | --- | --- |
| WAL insert 上层入口 | `src/backend/access/transam/xloginsert.c` | `XLogInsert()` | 组装 WAL record，并调用 `XLogInsertRecord()`。 |
| WAL record 插入共享 buffer | `src/backend/access/transam/xlog.c` | `XLogInsertRecord()` | 预留 WAL 空间、复制 record、处理 checkpoint redo 特殊 record。 |
| WAL insert head 推进 | `src/backend/access/transam/xlog.c` | `ReserveXLogInsertLocation()` | 推进 `Insert->CurrBytePos` 和 `Insert->PrevBytePos`。 |
| 等待 insert 完成 | `src/backend/access/transam/xlog.c` | `WaitXLogInsertionsToFinish()` | 写盘前等待目标范围内的并发插入完成。 |
| WAL flush | `src/backend/access/transam/xlog.c` | `XLogFlush()` | 确保 WAL 持久化；recovery 中转为推进 `minRecoveryPoint`。 |
| 启动主入口 | `src/backend/access/transam/xlog.c` | `StartupXLOG()` | 协调启动、恢复、结束恢复和初始化 WAL insert 状态。 |
| 创建 checkpoint | `src/backend/access/transam/xlog.c` | `CreateCheckPoint()` | 普通 checkpoint 主体。 |
| checkpoint/restartpoint 刷盘公共逻辑 | `src/backend/access/transam/xlog.c` | `CheckPointGuts()` | 刷 SLRU、buffer、fsync 队列和 2PC 状态。 |
| 记录安全 restartpoint 候选 | `src/backend/access/transam/xlog.c` | `RecoveryRestartPoint()` | replay 到 checkpoint record 时记录安全 restartpoint 候选。 |
| 创建 restartpoint | `src/backend/access/transam/xlog.c` | `CreateRestartPoint()` | recovery 中类似 checkpoint 的落盘点。 |
| 准备 recovery | `src/backend/access/transam/xlogrecovery.c` | `InitWalRecovery()` | 读取 checkpoint/control file/backup label，确定是否需要 recovery。 |
| 执行 WAL recovery | `src/backend/access/transam/xlogrecovery.c` | `PerformWalRecovery()` | 主 redo loop。 |
| 应用单条 WAL record | `src/backend/access/transam/xlogrecovery.c` | `ApplyWalRecord()` | 更新时间线和 replay 指针，调用 rmgr redo。 |
| replay 完成位置 | `src/backend/access/transam/xlogrecovery.c` | `GetXLogReplayRecPtr()` | 返回最后成功 replay 的 end LSN。 |
| 当前 replay 位置 | `src/backend/access/transam/xlogrecovery.c` | `GetCurrentReplayRecPtr()` | 包含正在应用中的 record。 |
| redo 函数注册表 | `src/include/access/rmgrlist.h` | `PG_RMGR(...)` | WAL record 的 resource manager 到 redo 函数的映射。 |
| 主机发送 WAL | `src/backend/replication/walsender.c` | `WalSndLoop()` / `XLogSendPhysical()` / `WalSndWaitForWal()` | walsender 等待并发送物理 WAL。 |
| 备机接收 WAL | `src/backend/replication/walreceiver.c` | `WalReceiverMain()` / `XLogWalRcvProcessMsg()` | walreceiver 主循环和消息处理。 |
| 备机写入 WAL | `src/backend/replication/walreceiver.c` | `XLogWalRcvWrite()` / `XLogWalRcvFlush()` | 把接收的 WAL 写入并 flush 到备机 `pg_wal`。 |

## 指针速查

| 指针/字段 | 所在结构 | 推进位置 | 含义 |
| --- | --- | --- | --- |
| `Insert.CurrBytePos` | `XLogCtlInsert` | `ReserveXLogInsertLocation()` | 已预留 WAL 空间的末端；近似 insert head。 |
| `Insert.PrevBytePos` | `XLogCtlInsert` | `ReserveXLogInsertLocation()` | 上一条已预留 record 的起点，用于 `xl_prev`。 |
| `Insert.RedoRecPtr` | `XLogCtlInsert` | `CreateCheckPoint()` / checkpoint redo special insert / `CreateRestartPoint()` | WAL insert 端看到的 redo 点，用于 full-page image 决策。 |
| `XLogCtl->RedoRecPtr` | `XLogCtlData` | `CreateCheckPoint()` / `CreateRestartPoint()` | info lock 保护的 redo 点副本。 |
| `LogwrtRqst.Write` | `XLogCtlData` | `XLogInsertRecord()` / `XLogFlush()` | 请求至少写到哪里。 |
| `LogwrtResult.Write` | backend-local/shared atomic result | `XLogWrite()` | 已经写到 OS/kernel cache 的位置。 |
| `LogwrtResult.Flush` | backend-local/shared atomic result | `XLogWrite()` | 已经 flush/sync 到持久化存储的位置。 |
| `replayEndRecPtr` | `XLogRecoveryCtlData` | `ApplyWalRecord()` replay 前 | 当前正在应用或已应用 record 的 end LSN。 |
| `lastReplayedEndRecPtr` | `XLogRecoveryCtlData` | `ApplyWalRecord()` replay 成功后 | 最后一条成功 replay 的 end LSN。 |
| `ControlFile->checkPoint` | `ControlFileData` | `CreateCheckPoint()` / `CreateRestartPoint()` | 最近 checkpoint record 自身的位置。 |
| `ControlFile->checkPointCopy.redo` | `ControlFileData` | `CreateCheckPoint()` / `CreateRestartPoint()` | 下次恢复的 redo 起点。 |

## 第一版仍可继续展开的问题

- 具体 rmgr redo 示例：例如 heap insert/update 的 WAL 生成和 replay 路径。
- checkpoint 触发条件：checkpointer 如何根据时间和 WAL 量触发 checkpoint/restartpoint。
- `minRecoveryPoint` 的推进细节：base backup、standby、`XLogFlush()` 在 recovery 中的特殊行为。
- WAL buffer 页面初始化、segment switch、`XLogWrite()` 内部写盘细节。
