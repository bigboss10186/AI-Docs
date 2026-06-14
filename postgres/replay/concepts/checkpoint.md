# Checkpoint

这篇先聚焦 checkpoint 的基本原理。replay、restartpoint、insert pointer 的细节会在这里保留接口，但更深入的内容可以继续拆到其他专题里。

## 1. Checkpoint 解决什么问题

PostgreSQL 修改数据页时，不要求数据页立刻刷到磁盘；它先保证 WAL 持久化。这样可以把随机的数据页写入延后、合并，但也带来一个问题：崩溃后必须从 WAL 中重新应用那些“已经写了 WAL、但数据页可能还没落盘”的修改。

checkpoint 的作用就是定期建立一个新的恢复边界：

- 在 checkpoint 之前的一批脏页和辅助状态被刷到磁盘。
- checkpoint record 被写入 WAL。
- `pg_control` 记录最近 checkpoint record 的位置和 checkpoint 内容。
- 下一次 crash recovery 可以从 checkpoint 中记录的 `redo` 位置开始，而不必从更早的 WAL 一直扫。

因此 checkpoint 的核心价值是缩短恢复时间，同时给 WAL 回收/保留提供边界。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：创建普通 checkpoint 的主流程。
- `src/backend/access/transam/xlog.c` 的 `CheckPointGuts()`：checkpoint 和 restartpoint 共用的刷盘逻辑。
- `src/backend/postmaster/checkpointer.c` 的 `CheckpointerMain()`：checkpointer 后台进程主循环，负责按时间、请求和 WAL 量触发 checkpoint。

## 2. Checkpoint 里最重要的几个位置

checkpoint 不是单个 LSN，而是一组互相关联的位置和状态：

- `checkPoint.redo`: 恢复需要从哪里开始回放。它是 checkpoint 里最关键的恢复下界。
- `ControlFile->checkPoint`: 最近 checkpoint record 自身在 WAL 中的位置。
- `ControlFile->checkPointCopy`: 最近 checkpoint record 的内容副本，其中包含 `redo`。
- `XLogCtl->Insert.RedoRecPtr`: WAL insert 路径看到的当前 redo 点，用于 full-page image 决策。
- `XLogCtl->RedoRecPtr`: info lock 保护的 redo 点副本，供其他路径读取。

为什么 `ControlFile->checkPoint` 和 `checkPoint.redo` 不是同一个东西？

因为 checkpoint record 是“描述 checkpoint 的记录”，而 `redo` 是“如果从这个 checkpoint 恢复，应该从哪里开始重放”。在线 checkpoint 期间二者通常不同：PostgreSQL 会先确定 redo 点，再花一段时间刷脏页，最后才写 checkpoint record。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：构造 `CheckPoint` 结构、设置 `checkPoint.redo`、写 checkpoint record。
- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：更新 `ControlFile->checkPoint` 和 `ControlFile->checkPointCopy`。
- `src/backend/access/transam/xlogrecovery.c` 的 `InitWalRecovery()`：启动恢复时读取 checkpoint record 和 `checkPoint.redo`。

## 3. 在线 checkpoint 的基本流程

在线 checkpoint 是系统正常运行时的 checkpoint，业务 backend 仍然可能继续产生 WAL。它的主流程可以理解为：

1. 准备 checkpoint，收集必要状态。
2. 确定新的 redo 点。
3. 释放 WAL insert 路径，让业务继续运行。
4. 刷共享 buffer、SLRU、复制槽、两阶段事务等持久状态。
5. 写 `XLOG_CHECKPOINT_ONLINE` checkpoint record。
6. flush checkpoint record。
7. 更新 `pg_control`。
8. 删除或回收不再需要的旧 WAL segment。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：整体流程。
- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()` 在线 checkpoint 分支：插入 `XLOG_CHECKPOINT_REDO`。
- `src/backend/access/transam/xlog.c` 的 `CheckPointGuts()`：刷脏状态。
- `src/backend/access/transam/xlog.c` 的 `RemoveOldXlogFiles()`：checkpoint 后清理旧 WAL。

### 为什么在线 checkpoint 要先写 `XLOG_CHECKPOINT_REDO`

在线 checkpoint 期间，其他 backend 还可以继续插入 WAL。checkpoint 要表达的是：“从某个 redo 点开始，之前需要持久化的状态已经被我刷盘覆盖了”。如果不先固定 redo 点，checkpoint 刷盘过程中产生的新 WAL 就可能和 checkpoint 的覆盖范围混在一起。

所以在线 checkpoint 会先插入一个特殊 WAL record：`XLOG_CHECKPOINT_REDO`。这个 record 的起始位置成为新的 `RedoRecPtr`。后续业务写 WAL 时，会看到新的 redo 点，并据此判断是否需要 full-page image。

对应代码：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：调用 `XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT_REDO)`。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：对 `XLOG_CHECKPOINT_REDO` 走特殊路径，独占 WAL insert locks，并更新 `RedoRecPtr = Insert->RedoRecPtr = StartPos`。

### 为什么更新 redo 点时要挡住 WAL insert

`RedoRecPtr` 会影响 full-page image 决策。checkpoint 推进 redo 点以后，某些数据页第一次被修改时可能需要写 full-page image，保证从新 redo 点恢复时页面有完整基准。

如果 backend 在旧 redo 点下组装 WAL record，但 checkpoint 同时把 redo 点推进了，就可能漏掉本该包含的 full-page image。所以代码在关键窗口持有 WAL insert locks，并且 `XLogInsertRecord()` 会在真正插入时重新检查 `RedoRecPtr` 和 `fullPageWrites`。如果发现状态变化会让上层重新组装 WAL record。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()`：独占 WAL insert locks 观察和推进 checkpoint 状态。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：持有 insert lock 时检查 `RedoRecPtr` 和 `fullPageWrites`。
- `src/backend/access/transam/xloginsert.c` 的 `XLogInsert()`：当 `XLogInsertRecord()` 返回 `InvalidXLogRecPtr` 时重新组装 WAL record。

## 4. Checkpoint 刷哪些东西

checkpoint 不只是刷 heap/index 数据页。`CheckPointGuts()` 会串起多个子系统的 checkpoint：

- relation map
- replication slots
- logical decoding snapshot builder
- logical rewrite heap
- replication origin
- CLOG/pg_xact
- commit timestamp
- subtrans
- MultiXact
- predicate locks
- shared buffers
- pending fsync requests
- two-phase state

为什么这些都要在 checkpoint 中处理？

因为 crash recovery 不只需要恢复数据页，也要恢复事务状态、子事务状态、MultiXact、两阶段提交、逻辑复制相关状态等。checkpoint 的承诺是“从这个 redo 边界恢复时，磁盘上的各种持久状态已经足够新”。如果只刷数据页，不刷这些辅助状态，恢复后的事务可见性和锁/复制状态可能不一致。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CheckPointGuts()`：checkpoint 刷盘公共逻辑。
- `src/backend/storage/buffer/bufmgr.c` 的 `CheckPointBuffers()`：刷共享 buffer。
- `src/backend/access/transam/clog.c` 的 `CheckPointCLOG()`：刷事务提交状态。
- `src/backend/access/transam/subtrans.c` 的 `CheckPointSUBTRANS()`：刷子事务状态。
- `src/backend/access/transam/multixact.c` 的 `CheckPointMultiXact()`：刷 MultiXact 状态。
- `src/backend/access/transam/twophase.c` 的 `CheckPointTwoPhase()`：处理两阶段事务状态。

## 5. Shutdown checkpoint 为什么不同

shutdown checkpoint 发生在数据库干净关闭时。此时已经没有并发普通 WAL 写入，所以它可以直接把当前 insert 位置作为 `checkPoint.redo`，并让 checkpoint record 自身承担恢复边界。

为什么 shutdown checkpoint 可以这样做？

因为 shutdown checkpoint 之后不允许再有新的 WAL 活动。如果 checkpoint 完成，控制文件状态会变成干净关闭；下次启动时通常不需要 crash recovery。如果 shutdown checkpoint 期间发现还有并发 WAL 活动，代码会直接 PANIC，因为这会破坏“checkpoint 后没有新 WAL”的前提。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()` shutdown 分支：计算 `curInsert`，设置 `checkPoint.redo`，并更新 `RedoRecPtr`。
- `src/backend/access/transam/xlog.c` 的 `CreateCheckPoint()` shutdown 校验：如果 shutdown checkpoint 期间还有并发 WAL 活动，会 PANIC。

## 6. Checkpoint 与 recovery 的关系

启动恢复时，PostgreSQL 会从 `pg_control` 或 `backup_label` 找到 checkpoint record，读取其中的 `CheckPoint` 内容，再从 `checkPoint.redo` 开始寻找和回放 WAL record。

为什么 recovery 不是从 checkpoint record 后面开始？

因为在线 checkpoint 的 `redo` 位置可能早于 checkpoint record 自身。checkpoint record 写入时，checkpoint 已经刷了很多脏页；但 crash recovery 的安全起点是 checkpoint 开始时确定的 redo 点，而不是 checkpoint record 写完的位置。恢复必须从 `checkPoint.redo` 开始，才能覆盖 checkpoint 期间的并发变化和边界条件。

源码坐标：

- `src/backend/access/transam/xlogrecovery.c` 的 `InitWalRecovery()`：读取 checkpoint record，确定 `RedoStartLSN`。
- `src/backend/access/transam/xlogrecovery.c` 的 `PerformWalRecovery()`：从 redo 点定位第一条要回放的 WAL record，并进入 redo loop。
- `src/backend/access/transam/xlogrecovery.c` 的 `ApplyWalRecord()`：应用单条 WAL record。

## 7. Checkpoint 与 restartpoint 的关系

在 standby/recovery 中，系统正在读取并应用 WAL，不能像 primary 那样随意创建普通 checkpoint。PostgreSQL 使用 restartpoint 来达到类似目的：当 standby 已经 replay 到 primary 产生的安全 checkpoint record 后，checkpointer 可以把当前状态刷盘，并更新控制文件，让未来恢复从更近的位置开始。

为什么 standby 的 restartpoint 不能随便选一个 LSN？

因为 restartpoint 必须对应 WAL 流中已经存在的、由 primary 生成的 checkpoint 边界。standby 不能自己发明一个新的普通 checkpoint record 来改变 WAL 历史；它只能在 replay 到安全 checkpoint record 后，把这个点作为本地重启恢复的边界。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `RecoveryRestartPoint()`：startup process replay 到 checkpoint record 时，判断是否能记录为安全 restartpoint 候选。
- `src/backend/access/transam/xlog.c` 的 `CreateRestartPoint()`：checkpointer 实际创建 restartpoint。
- `src/backend/access/transam/xlog.c` 的 `CreateRestartPoint()`：把 shared `RedoRecPtr` 推进到 `lastCheckPoint.redo`，调用 `CheckPointGuts()`，并更新 control file 和 `minRecoveryPoint`。

## 8. 后续适合继续补充的“为什么”

- 为什么 checkpoint 完成后才能回收某些旧 WAL segment？
- 为什么 full-page writes 和 checkpoint redo 点强相关？
- 为什么 checkpoint 要等待处于 commit critical section 的事务？
- 为什么 `CheckPointTwoPhase()` 被故意放在 `CheckPointGuts()` 的后半段？
- 为什么 restartpoint 有时会被跳过？
- 为什么 `minRecoveryPoint` 在 standby/backup 场景中特别重要？

