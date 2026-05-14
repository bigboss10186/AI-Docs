# WAL Insert Head/Tail And Flush Progress

## 1. 先对齐术语

PostgreSQL 源码里不直接使用 “insert head” 和 “insert tail” 这两个术语。为了阅读方便，可以做如下映射：

- insert head: `XLogCtl->Insert.CurrBytePos`，表示已经预留出去的 WAL 可用字节末端。下一条 WAL record 会从这个位置开始预留。
- previous record pointer: `XLogCtl->Insert.PrevBytePos`，表示上一条已预留 WAL record 的起点，用于填入下一条 record header 的 `xl_prev`。
- finished insert boundary: `XLogCtl->logInsertResult` 和 `WaitXLogInsertionsToFinish()` 计算出的边界，表示 WAL record 已经从 backend 私有结构拷贝到共享 WAL buffer 的进度。
- write/flush progress: `XLogCtl->LogwrtRqst` 表示请求写到哪里，`LogwrtResult` 表示已经写/flush 到哪里。

如果你说的 “insert tail” 是“最早还没完成的 WAL 插入”，它不是一个单独字段，而是由多个 `WALInsertLock.insertingAt` 的状态动态推导出来。`WaitXLogInsertionsToFinish()` 会扫描这些锁，把可安全写出的边界从已预留 head 往回收缩到所有相关插入都完成的位置。

## 2. WAL 插入分两步

`XLogInsert()` 是上层入口，先把调用者注册的数据和 buffer references 组装成 WAL record，再交给 `XLogInsertRecord()` 放入 WAL buffer。

源码坐标：

- `src/backend/access/transam/xloginsert.c` 的 `XLogInsert(RmgrId rmid, uint8 info)`。
- `src/backend/access/transam/xloginsert.c` 的 `XLogInsert()`：读取 `RedoRecPtr` 和 `doPageWrites`。
- `src/backend/access/transam/xloginsert.c` 的 `XLogInsert()`：通过 `XLogRecordAssemble(...)` 组装 WAL record。
- `src/backend/access/transam/xloginsert.c` 的 `XLogInsert()`：调用 `XLogInsertRecord(...)`。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord(...)` 是共享 WAL buffer 插入主体。

`XLogInsertRecord()` 的核心分两步：

1. Reserve：在 `Insert->CurrBytePos` 上原子地预留 WAL 空间，得到 `StartPos`、`EndPos`，并设置 record header 的 `xl_prev`。
2. Copy：把 WAL record 内容复制到预留的 WAL buffer 页面中。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：源码注释明确描述 reserve/copy 两个步骤。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：普通 WAL record 获取一个 WAL insert lock。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：调用 `ReserveXLogInsertLocation(...)`。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：调用 `CopyXLogRecordToWAL(...)`。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：释放 WAL insert lock，表示本次插入完成。

## 3. insert head 如何推进

`ReserveXLogInsertLocation()` 是 WAL insert head 推进的关键点。它在 `insertpos_lck` 下做很短的临界区：

1. 读取 `Insert->CurrBytePos` 作为 `startbytepos`。
2. `endbytepos = startbytepos + size`。
3. 读取 `Insert->PrevBytePos` 作为 `prevbytepos`。
4. 设置 `Insert->CurrBytePos = endbytepos`。
5. 设置 `Insert->PrevBytePos = startbytepos`。
6. 释放锁后把 byte position 转成真正的 `XLogRecPtr`。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `ReserveXLogInsertLocation(...)`。
- `src/backend/access/transam/xlog.c` 的 `ReserveXLogInsertLocation()`：解释为什么内部用 “usable byte positions” 而不是直接用 `XLogRecPtr`。
- `src/backend/access/transam/xlog.c` 的 `ReserveXLogInsertLocation()`：推进 `CurrBytePos` 和 `PrevBytePos`。
- `src/backend/access/transam/xlog.c` 的 `ReserveXLogInsertLocation()`：转换出 `StartPos`、`EndPos`、`PrevPtr`。

这意味着 `CurrBytePos` 推进只表示空间已预留，不代表 WAL record 内容已经完整复制到 WAL buffer，也不代表已经写盘。

## 4. `PrevBytePos` 推进的意义

`PrevBytePos` 保存上一条 record 的起点。预留新 record 时，代码把旧 `PrevBytePos` 转成 `PrevPtr`，填入新 record header 的 `xl_prev`；随后把 `PrevBytePos` 更新为本 record 的 `startbytepos`。

这使 WAL record 形成向后的链，用于校验、遍历和处理一些恢复场景。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `XLogCtlInsert`：`CurrBytePos` 和 `PrevBytePos` 的结构定义及注释。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：`ReserveXLogInsertLocation()` 会设置 `xl_prev`。
- `src/backend/access/transam/xlog.c` 的 `ReserveXLogInsertLocation()`：`prevbytepos` 取旧值，随后 `PrevBytePos` 更新为当前 record 起点。

## 5. 从预留到可写盘：动态 tail

WAL writer 或提交路径不能只看 `CurrBytePos` 就直接写到那里，因为有些 backend 可能已经预留了空间，但还没把 record 内容复制完。

PostgreSQL 用 `WALInsertLock` 跟踪正在插入的 backend：

- 每个 inserter 持有一个 WAL insert lock。
- `insertingAt` 可以标记该 inserter 已经推进到哪里。
- 写盘前调用 `WaitXLogInsertionsToFinish(upto)`，等待所有会影响目标写盘范围的插入完成。
- 函数返回的 `finishedUpto` 才是可安全写出的边界。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `WALInsertLock`：设计说明。
- `src/backend/access/transam/xlog.c` 的 `WaitXLogInsertionsToFinish(XLogRecPtr upto)`。
- `src/backend/access/transam/xlog.c` 的 `WaitXLogInsertionsToFinish()`：读取当前 `CurrBytePos`，得到已预留边界 `reservedUpto`。
- `src/backend/access/transam/xlog.c` 的 `WaitXLogInsertionsToFinish()`：`finishedUpto` 先设为已预留 head，然后根据未完成插入向后收缩。
- `src/backend/access/transam/xlog.c` 的 `WaitXLogInsertionsToFinish()`：扫描各个 WAL insert lock，等待必要的插入完成。

## 6. 写入和 flush 如何推进

`XLogFlush(record)` 保证给定 LSN 之前的 WAL 已经 flush 到磁盘。它不会盲目 flush 目标点，而是会：

1. 如果当前处于 redo/recovery 且不允许插入 WAL，则转为更新 `minRecoveryPoint`。
2. 如果目标已经 flush，直接返回。
3. 合并全局 `LogwrtRqst.Write`，尽量多写一些。
4. 调 `WaitXLogInsertionsToFinish()` 等待目标范围内的 WAL insert 完成。
5. 获取 `WALWriteLock`。
6. 调 `XLogWrite()` 写和刷 WAL。
7. 唤醒 walsender。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `XLogFlush(XLogRecPtr record)`。
- `src/backend/access/transam/xlog.c` 的 `XLogFlush()`：recovery 中不写 WAL，改为 `UpdateMinRecoveryPoint(record, false)`。
- `src/backend/access/transam/xlog.c` 的 `XLogFlush()`：合并写请求并等待相关 WAL 插入完成。
- `src/backend/access/transam/xlog.c` 的 `XLogFlush()`：获取 `WALWriteLock` 并复查是否已经被其他 backend flush。
- `src/backend/access/transam/xlog.c` 的 `XLogFlush()`：把 `insertpos` 作为写/flush 目标交给 `XLogWrite()`。

## 7. 观察 insert 进度的接口

两个函数都读取 `Insert->CurrBytePos`，区别是转换方式不同：

- `GetXLogInsertRecPtr()` 返回 latest WAL insert pointer。
- `GetXLogInsertEndRecPtr()` 返回 latest WAL record end pointer。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `GetXLogInsertRecPtr(void)`。
- `src/backend/access/transam/xlog.c` 的 `GetXLogInsertEndRecPtr(void)`。

## 8. insert 推进与 checkpoint 的关系

checkpoint 推进 redo 点时，需要和 WAL insert 路径协调：

- 在线 checkpoint 要确定一个新的 redo 点。
- 确定 redo 点时，不能让并发插入看到旧状态后又把页面修改算错。
- 所以 `CreateCheckPoint()` 会独占 WAL insert locks；在线 checkpoint 再写 `XLOG_CHECKPOINT_REDO`，由 `XLogInsertRecord()` 在特殊路径中更新 `Insert->RedoRecPtr`。
- 后续普通 WAL insert 在持有 insert lock 后检查 `RedoRecPtr` 是否变化；如果 full-page image 决策受影响，会返回 `InvalidXLogRecPtr` 让上层重新组装 record。

源码坐标：

- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：持有 insert lock 会保护 `RedoRecPtr` 和 `fullPageWrites` 不变。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：普通 insert 发现 redo/full-page 状态变化时，可能要求重新组装 WAL record。
- `src/backend/access/transam/xlog.c` 的 `XLogInsertRecord()`：checkpoint redo record 的特殊插入路径更新 shared/local `RedoRecPtr`。
