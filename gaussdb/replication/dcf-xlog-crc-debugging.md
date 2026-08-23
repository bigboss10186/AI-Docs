# openGauss DCF 模式下 XLog CRC 偶现失败调试手册

## 文档信息

| 字段 | 内容 |
| --- | --- |
| 主题 | openGauss DCF / WAL record CRC / core 定位 |
| 类型 | 排障复盘 |
| 状态 | 可用 |
| 创建时间 | 2026 年 8 月 23 日 |
| 更新时间 | 2026 年 8 月 23 日 |

## 目标

这篇文档用于定位下面这类问题：

- 流复制场景中已经稳定运行的 WAL record CRC 校验逻辑，接入 DCF 发送端或接收端后偶现失败；
- 发送端在 DCF entry 从内存 cache 取不到、需要从磁盘恢复 WAL body 时进行校验；
- 接收端在 `walRcvWrite()` 附近根据 `buf/nbytes/start LSN` 增量解析 WAL record 并校验；
- 同一个故障出现后，接收端进程重启仍然重复 core；
- 两个备机可能在同一个 WAL 位置同时 core；
- 私有 DCF 分支采用 header-only 优化，只持久化 entry 引用信息，发送前再从本地 WAL 文件拼装 body。

目标不是一开始就打印所有状态，而是让下一次复现至少能回答三个问题：

1. 主机写入 DCF、发送端重建和备机接收的物理 WAL 字节是否一致？
2. 自定义校验器识别出的 record 起点、固定 header 和逻辑长度是否正确？
3. openGauss 原生 `XLogReader` 是否认可同一个 record？

## 现象

当前已知现象包括：

- CRC 失败是偶现的，暂时没有稳定的业务触发条件；
- 第一次 core 后，接收端重启可能持续 core；
- 两个备机可能对同一个 XLog 位置同时失败；
- 出错时解析出的 `xl_tot_len` 可能超过 10000；
- 出错版本中 record body 看起来基本正常，但保存、跨页拼接出来的固定 `XLogRecord` header 值可疑；
- 私有 DCF 代码中还存在 entry checksum 不一致的日志，但当前逻辑只打印，没有阻止发送。

## 当前结论

- `xl_tot_len` 超过 10000 本身正常。它表示整个逻辑 WAL record 的长度，不是固定 record header 的长度；单条 record 可以跨多个 8 KiB WAL page。
- 一个 DCF entry/index 对应一次 `dcf_write()` 的连续物理 WAL 区间，不对应一条 WAL record。一个 entry 可以包含多条 record，一条 record 也可以跨多个 entry。
- 两个备机在同一个 record LSN 上失败，说明问题具有确定性；它降低了节点私有线程状态、随机内存残留和偶发竞争的概率，但仍然无法单独区分：
  - leader 重建出同一份错误 WAL，并发给两个备机；
  - 两个备机使用相同的校验状态机，对同一种合法 WAL 布局产生相同误判。
- 重启后持续 core 不能单独证明 WAL 已损坏。DCF 只有在 apply 回调成功返回后才推进 applied index；回调内部 core 会让相同 DCF index 在重启后再次投递，形成稳定的“毒 entry 重放”。
- 如果备机原生 redo 已经正常越过失败 record，并且 `XLogReader` 没有报告 CRC 错误，则物理 WAL 基本可信，应优先定位自定义 checker。
- 如果 DCF entry checksum 的保存值确实来自原始 `dcf_write()` payload，而回读值来自最终拼装的完整 WAL payload，那么 checksum 不一致说明 header-only body 重建路径已经改变了字节；但必须先确认两次 checksum 使用了完全相同的 `buf/len` 语义。

## 四个观测点

把一段相同的物理 WAL 区间记为 `[start_lsn, end_lsn)`，沿链路建立四个观测点：

```text
A. openGauss 调用 dcf_write() 前
   原始 buf / nBytes / end LSN / physical hash

B. DCF leader cache miss 后
   从本地 WAL 文件重建出的 send_buf / send_len / end LSN / physical hash

C. DCF follower ReceiveLogCbFunc()
   ENTRY_BUF / ENTRY_SIZE / key / physical hash

D. walreceiverwriter / 原生 XLogReader
   实际写盘字节 / 自定义 record CRC / 原生 record CRC
```

这里需要区分两类 checksum：

- entry physical hash/checksum：对完整物理 WAL 区间逐字节计算，包含 WAL page header 和 padding，不要求 record 对齐；
- WAL record CRC：对一条逻辑 record 计算，排除物理 WAL page header、对齐 padding 和 `xl_crc` 字段自身，计算顺序为 record data 在前、固定 header 在后。

entry checksum 更适合判断“字节在哪一层发生变化”，record CRC 更适合判断“逻辑 WAL record 是否有效”。

## 为什么重启后会持续 core

DCF follower apply 的关键顺序是：

```text
rep_apply_proc(index N)
  → stg_get_entry(index N)
  → ReceiveLogCbFunc(...)
      → XLogWalRcvReceive(...)
          → walRcvWrite(...)
              → 自定义 CRC 校验
                  → core
  → 回调成功返回后才 stg_set_applied_index(index N)
```

如果 core 发生在 `walRcvWrite()` 的校验路径中：

- `ReceiveLogCbFunc()` 无法正常返回；
- `DcfUpdateAppliedRecordIndex()` 也可能尚未执行；
- DCF 的 applied index 不会推进；
- 重启后仍然从同一个 index 开始 apply；
- 相同的真实坏字节或相同的确定性 checker bug 会再次 core。

因此，持续 core 主要排除了“旧进程 TLS 中间态残留”，但不能区分真实坏 WAL 与确定性误校验。

比较连续两次 core 时，先看：

```text
DCF index
entry start/end LSN
record start LSN
xl_tot_len
stored xl_crc
calculated crc
```

- 所有字段完全相同：同一个毒 entry/record 的确定性问题；
- stored CRC 相同，但 calculated CRC 每次不同：未初始化内存、状态未完整清理、buffer 生命周期或并发覆盖更可疑；
- 每次失败 LSN 不同，但布局特征相同：可能是一类 page/segment/continuation 边界没有适配。

## 最小诊断方案

第一轮不要打印所有 page 和 record，只增加三组信息。

### 1. DCF entry 级别

```text
thread_id
stream_id
node_id
index
entry_type
source = cache / disk
entry_start_lsn
entry_end_lsn
original_len
rebuilt_len
stored_entry_checksum
rebuilt_entry_checksum
cross_page
cross_segment
```

已经确认只对 `ENTRY_TYPE_LOG` 调用 WAL CRC 后，`ENTRY_TYPE_CONF` 不应进入校验器，也不应因为它占用了一个 DCF index 就清理 WAL CRC 状态。

### 2. checker 输入和 reset

```text
input_start_lsn
input_end_lsn
nbytes
previous_expected_lsn
是否发生 discontinuity reset
reset 原因：gap / overlap / retry / rewind / epoch change
当前状态：header / data / continuation / padding / resync
```

发送端只在 disk miss 时校验，意味着 checker 看到的不是完整发送流。cache hit entry 会被跳过，因此下一个 disk entry 到达时很可能与旧 `expected_lsn` 不连续。此时必须完整 reset，并从可信 WAL page 边界重新同步。

### 3. CRC 首次失败

```text
record_start_lsn
record_start_lsn % XLOG_BLCKSZ
xl_tot_len
xl_term
xl_xid
xl_prev
xl_info
xl_rmid
xl_bucket_id
stored_xl_crc
calculated_crc
fixed_header_bytes_saved
logical_record_bytes_saved
涉及的最近几个 DCF index
```

仅凭 `xl_tot_len = 10000+` 不能判断 header 错误。更有效的 header 合法性检查是：

- `SizeOfXLogRecord <= xl_tot_len < XLogRecordMaxSize`；
- `xl_rmid <= RM_MAX_ID`；
- 顺序读取时 `xl_prev` 等于上一条 record 的起点；
- 随机接入时至少满足 `xl_prev < record_start_lsn`；
- record 起点不是 WAL page header 或 alignment padding。

## 用小型循环缓冲代替大量日志

偶现问题不适合长期打印每个 record。可以在线程局部状态中保存最近 4 到 8 个事件，CRC 失败时一次性输出，或者直接从 core 中读取。

示意结构：

```c
#define CRC_DEBUG_HISTORY_SIZE 8

typedef struct CrcDebugItem {
    uint64 index;
    XLogRecPtr input_start;
    XLogRecPtr input_end;
    XLogRecPtr record_start;
    uint32 input_len;
    uint32 xl_tot_len;
    uint32 fixed_header_saved;
    uint32 logical_record_saved;
    uint32 stored_crc;
    uint32 calculated_crc;
    bool from_disk;
    bool reset;
} CrcDebugItem;

THR_LOCAL volatile CrcDebugItem g_crc_debug_history[CRC_DEBUG_HISTORY_SIZE];
THR_LOCAL volatile uint32 g_crc_debug_pos;
```

`volatile` 主要是为了降低调试变量被编译器完全优化掉的概率，不用于线程同步。每个 append/receiver 线程仍然维护自己的槽。

还可以固定保存少量现场字节：

```c
THR_LOCAL volatile char g_failed_header[SizeOfXLogRecord];
THR_LOCAL volatile char g_failed_first_bytes[64];
THR_LOCAL volatile char g_failed_last_bytes[64];
```

不要默认 dump 完整 record。record 最大可以接近 1 GiB，优先保存固定 header、首尾字节和逐 page hash；只有在受控测试环境中才保存完整 record 或相关 WAL segment。

## 利用 core 文件定位

### core 能提供什么

如果 CRC 不一致后直接 `Assert`、`abort` 或 `PANIC`，core 通常可以保留：

- 当前调用栈和崩溃行；
- 当前校验函数的参数；
- TLS、全局变量和仍然存活的 heap buffer；
- 保存的残留 header；
- 当前 record 累计状态；
- 当前 `buf/nbytes/start LSN`，前提是对应内存页被 core 收录。

core 不是执行历史，不能自动恢复已经处理、释放或覆盖的前几个 record。要观察历史，需要上面的循环诊断缓冲。

### 使用条件

- core 对应的 `gaussdb` 二进制必须完全匹配；
- 保留匹配的 debug symbol；
- 二进制不要 strip，最好使用 `-g`；
- 高优化构建可能出现局部变量 `<optimized out>`；
- 共享内存是否进入 core 取决于操作系统和 core dump 配置，因此关键字节最好额外复制到 TLS 诊断结构。

### 基本 GDB 操作

```gdb
gdb /path/to/gaussdb /path/to/core

thread apply all bt full
thread <walreceiverwriter-thread-number>
frame <crc-check-frame-number>
info args
info locals
```

检查状态：

```gdb
p record_start_lsn
p saved_header_len
p logical_record_bytes_saved
p expected_lsn
p stored_crc
p calculated_crc
p g_crc_debug_history

x/32bx saved_header_buf
x/64bx current_buf
```

如果完整逻辑 record buffer 仍然存在，可以从 core 导出：

```gdb
dump binary memory /tmp/failed-record.bin \
    record_buf \
    record_buf + record_total_len
```

如果 core 只能看到固定 header，也已经足够检查 `xl_tot_len/xl_prev/xl_rmid/xl_crc` 是否来自正确的 record 起点。

## 固定 XLogRecord header 跨页检查

openGauss 插入 WAL 时只保证 record 起始页至少能放下前四字节 `xl_tot_len`：

```c
Assert(freespace >= sizeof(uint32));
```

它不保证整个固定 `XLogRecord` header 都在同一页。因此合法布局可以是：

```text
record start page 尾部:
  [xl_tot_len][固定header的前半部分]

next WAL page:
  [short/long page header]
  [固定XLogRecord header剩余部分]
  [record body]
```

这个保证只针对物理 WAL page。任意 DCF entry、walreceiver ring 或 `walRcvWrite()` 输入边界仍然可以把 `xl_tot_len` 的四个字节拆成两次调用。此时应该暂存不足四字节的输入；下一次输入如果仍位于同一物理 page，就直接继续拼接，不能因为“换了一个 buf”就跳过 page header。

用下面的条件快速识别固定 header 跨页：

```c
uint32 page_remain = XLOG_BLCKSZ - record_start_lsn % XLOG_BLCKSZ;

page_remain >= sizeof(uint32) &&
page_remain < SizeOfXLogRecord
```

如果故障 record 都满足这个条件，优先检查保存 header 的逻辑是否把下一页的物理 page header 错当成了 record header。

正确的物理拼接模型是：

```text
第一页复制 page_remain 个 record 字节
到达物理 page boundary
解析并跳过 short/long WAL page header
再复制 SizeOfXLogRecord - page_remain 个 record header 字节
```

下一页使用 long 还是 short header 取决于物理 LSN：

```c
page_lsn % XLogSegSize == 0
    ? SizeOfXLogLongPHD
    : SizeOfXLogShortPHD
```

必须区分：

```text
新的回调/buf 开始
    不一定有 WAL page header

physical_lsn % XLOG_BLCKSZ == 0
    才表示进入新的物理 WAL page
```

另一个常见错误是用：

```text
record_end_lsn = record_start_lsn + xl_tot_len
```

这只在 record 没有跨物理 WAL page 时成立。`xl_tot_len` 不包含中间插入的 short/long page header，跨页时必须按逻辑字节计数并单独推进物理 LSN。

建议分别维护：

```text
fixed_header_bytes_saved
logical_record_bytes_saved
current_physical_lsn
expected_input_lsn
```

不要使用一个“残留 record 长度”同时表达固定 header 已保存长度、整个逻辑 record 已保存长度和物理 LSN 推进量。

## continuation 和重同步检查

在物理 WAL page 起点需要先解析 page header：

```text
xlp_magic
xlp_info
xlp_pageaddr
xlp_rem_len
```

如果设置 `XLP_FIRST_IS_CONTRECORD`：

- page header 后首先是上一条 record 的 continuation；
- continuation 没有新的 `xl_tot_len`；
- `xlp_rem_len` 表示从本页开始仍然剩余的逻辑 record 字节数；
- continuation 跨下一页时，需要继续跳过下一页 page header；
- continuation 结束后，还需要处理 `MAXALIGN` padding，才能到达下一条可信 record 起点。

状态 reset 后如果输入从 page 中间开始，当前字节可能是：

- record header；
- record body；
- alignment padding。

仅凭字节内容无法可靠判断。安全策略是放弃当前 page 剩余部分，从下一个可验证的 page header 开始重新同步。这样可能少校验一条 record，但不会把 body 中的四个字节误当成 `xl_tot_len`。

## `original_wal_len` 的来源

### 开源标准路径

标准 openGauss/DCF 中，`original_wal_len` 不是新增推导值，它就是 `XLogWritePaxos()` 传给 `dcf_write()` 的 `nBytes`。长度沿调用链原样传递：

```text
XLogWritePaxos
  nBytes
    ↓
dcf_write(buffer, length = nBytes, key = WriteLSN)
    ↓
rep_write(buffer, length, key)
    ↓
stg_append_entry(..., size = length, key)
    ↓
stream_append_entry(..., size)
    ↓
IO_BUF_SIZE(entry) = size
    ↓
ENTRY_SIZE(entry) = original_wal_len
```

同时：

```c
WriteLSN = LogwrtPaxos->Write + nBytes;
dcf_write(1, from, nBytes, WriteLSN, &paxosIdx);
```

因此标准完整 payload 版本满足：

```text
original_wal_len = nBytes = ENTRY_SIZE(entry)
entry_end_lsn    = ENTRY_KEY(entry) = WriteLSN
entry_start_lsn  = entry_end_lsn - original_wal_len
```

### 私有 header-only 路径

header-only 优化后必须区分：

```text
original_wal_len：最初 dcf_write() 的 WAL payload 长度
stored_reference_len：DCF segment 文件中实际保存的引用 header 长度
assembled_send_len：发送前从本地 WAL 重建出的最终 payload 长度
```

正确关系应当是：

```text
original_wal_len == assembled_send_len
stored_reference_len 可以更小
```

私有实现可能采用以下任一种方式保存 `original_wal_len`：

- 保留 `IO_BUF_SIZE/ENTRY_SIZE` 的原始逻辑语义，只让 segment 物理存储长度使用另一个字段；
- 在自定义引用 header 中增加 `original_size/wal_len`；
- 同时保存 start/end LSN，通过 `end_lsn - start_lsn` 恢复长度；
- 根据相邻 entry key 推导 start LSN，但这种方式对截断、缺 entry 和配置 entry 更敏感。

不能仅凭字段名判断。核对非公开版本时，应从最初的 `dcf_write(length)` 开始，逐层跟踪：

```text
dcf_write.length
  → rep_write.length
  → stream_append_entry.size
  → 持久化引用header中的长度字段
  → segment_get_entry读取时的物理分配长度
  → body重建使用的WAL长度
  → stg_get_entry返回后的ENTRY_SIZE
  → rep_encode_one_log/mec_put_bin最终发送长度
  → 内核CRC回调长度
```

对同一个 `index/key`，建议临时打印上述关键阶段的长度。内核 CRC 回调必须使用与最终 `mec_put_bin()` 完全相同的 WAL `buf/len`，而不是 DCF segment 的物理存储长度。

## 为什么标准 DCF entry 不拆开四字节 `xl_tot_len`

这个结论依赖当前标准 `XLogWritePaxos()` 的全部截断来源，而不只是 1 MiB 上限。

每轮先计算：

```c
WriteTotal = PaxosWriteRqst - LogwrtPaxos->Write;

CriticalNBytes =
    从当前WAL cache位置到cache数组末尾的连续物理字节数;

nBytes = Min(WriteTotal,
             Min(MaxSendSizeBytes, CriticalNBytes));
```

`nBytes` 的实际限制来源有三种：

| 限制来源 | 当前代码中的 entry end | 是否可能拆四字节 `xl_tot_len` |
| --- | --- | --- |
| `WriteTotal` 最小 | `PaxosWriteRqst` | 当前两个调用点传入完整 WAL copy status 的 `endLSN`，或者大 record copy 到达的物理 page boundary |
| `MaxSendSizeBytes` 最小 | 显式减去 `tmpLsn % XLOG_BLCKSZ`，回退到物理 page boundary | 不会；record 起始页保证至少容纳四字节 |
| `CriticalNBytes` 最小 | WAL cache 数组末尾；根据公式，物理 LSN offset 回到 page boundary | 不会；同样落在物理 page boundary |

当前两个 `XLogWritePaxos()` 调用来源是：

1. WAL background flush 使用 `curr_entry_ptr->endLSN`。它是已经完成 WAL copy 的状态项末尾，不是任意输入 buffer 位置；
2. `XLogSelfFlushWithoutStatus()` 在复制超大 record、WAL buffer 环回时传入 `currPos`。调用发生前已把当前 page 的 `freespace` 全部复制完，因此 `currPos` 位于物理 WAL page boundary。

WAL insert 端同时保证：

```c
Assert(freespace >= sizeof(uint32));
```

也就是一条新 record 开始时，其起始 page 一定可以完整放下四字节 `xl_tot_len`。于是标准 entry end 只有两类可信位置：

```text
完整record/copy-status末尾
物理WAL page boundary
```

前者不会截断 record；后者即使截断固定 `XLogRecord` header，也已经包含完整的四字节 `xl_tot_len`。所以标准路径下：

```text
xl_tot_len四字节不会跨两个DCF entry
固定XLogRecord header的剩余部分仍可能跨两个DCF entry
record body可以跨多个DCF entry
```

例如：

```text
entry N尾部，同时也是page尾部:
  [完整xl_tot_len][固定header前半部分]

entry N+1:
  [short/long page header]
  [固定header剩余部分]
  [record body]
```

这项保证只适用于“一个回调输入等于一个完整原始 DCF entry”的发送端。它不适用于：

- walreceiver ring/walRcvWrite 的任意连续区间；
- MEC fragment；
- body 重建过程中的单次 `pread`；
- 私有代码新增的任意字节限流或重新切片。

发送端内核回调如果放在 `stg_get_entry()` 完成全部 body 拼装之后、`rep_encode_one_log()` 之前，并且一次传入完整 `ENTRY_BUF/ENTRY_SIZE`，才可以使用这项 entry 边界保证。

### 非公开版本核对清单

在非公开版本中需要重新确认：

1. `XLogWritePaxos()` 的 `nBytes` 是否仍然只取 `WriteTotal/MaxSendSizeBytes/CriticalNBytes` 三者最小值；
2. 命中任意发送大小上限时，是否仍然回退到 `XLOG_BLCKSZ` page boundary；
3. 所有 `XLogWritePaxos()` 调用点传入的是 record/copy-status end，还是新增了任意 LSN；
4. `dcf_write()` 是否仍然一次调用创建一个 index，内部有没有按更小大小重新切 entry；
5. header-only 持久化是否保留 `original_wal_len`，有没有把它替换成引用 header 长度；
6. `stg_get_entry()` 返回前是否已经拼出完整原始 WAL payload；
7. 内核 CRC 回调是一整个 assembled entry 调用一次，还是每个 `pread`/拼装片段调用一次；
8. `rep_encode_one_log()` 最终发送的 `buf/len/key` 是否与内核 CRC 回调完全相同。

只要其中一项发生变化，就不能继续假设 `xl_tot_len` 不会跨发送端回调边界。

## header-only DCF 优化检查

私有实现如果只持久化引用 header，发送时从本地 WAL 文件拼 body，应同时维护两组长度：

```text
stored_reference_len
original_wal_len
```

CRC 回调必须使用最终发送的三元组：

```text
send_buf
send_len = original_wal_len
end_lsn = original entry key
```

并满足：

```text
start_lsn = end_lsn - send_len
```

不能使用：

- DCF 引用 header 长度；
- DCF 磁盘实际落盘长度；
- 临时 buffer 容量；
- 只读取到当前 WAL segment 末尾的长度。

即使长度正确，body 仍可能错误。磁盘恢复路径还要检查：

1. entry 是否跨 `XLogSegSize`，跨文件时是否逐段读取；
2. 每次 `pread` 的请求长度和实际返回长度是否完全相同；
3. 是否使用正确 timeline、segment 编号和 segment 内 offset；
4. 本地 WAL write/flush LSN 是否已经覆盖 `entry_end_lsn`；
5. 被引用的 WAL segment 是否已被 recycle 或复用；
6. 临时拼装 buffer 是否全部覆盖，尾部是否残留旧数据；
7. CRC/checksum 校验失败后是否仍继续调用 `rep_encode_one_log()`。

标准 openGauss 路径先执行 `XLogWritePaxos()`，再执行本地 `XLogFlushCore()`。header-only 优化必须额外保证：DCF cache 中 body 被回收、需要从 `pg_xlog` 回读时，对应物理 WAL 已经可以安全读取。

## DCF checksum 如何使用

开源 DCF 创建 entry 时会对完整 payload 计算 `ENTRY_DATA_CHKSUM`。标准磁盘读取路径在 checksum 不一致时把 entry 标记为无效，而不是继续发送。

私有 header-only 实现如果只打印不一致，需要先确认：

```text
stored checksum 的计算 buf/len
rebuilt checksum 的计算 buf/len
checksum 是否在剥离 body 前后被重新计算
ENTRY_SIZE 在两次计算中的语义
```

只有下面条件全部成立时，DCF checksum mismatch 才能直接证明 body 发生变化：

```text
stored_checksum = checksum(original dcf_write payload, original_nbytes)
rebuilt_checksum = checksum(final send payload, final_send_len)
original_nbytes == final_send_len
```

发生 mismatch 后应优先停止当前 entry 的发送并保持 `NEXT_INDEX` 不推进。可以重新读取并重试；持续失败时应升级为缺日志、重建或明确故障，不能把已知不一致的 payload 继续传播给备机。

## 判定矩阵

| 证据 | 更可能的结论 |
| --- | --- |
| 两个备机在相同 record LSN 上得到相同 stored/calculated CRC | 确定性公共问题，不像随机 TLS 残留 |
| 主机原始 WAL、发送重建和两个备机物理 hash 完全相同，原生 XLogReader 通过 | 自定义 checker 的 record/page 状态机错误 |
| 两个备机物理 hash 相同，但与主机原始 WAL 不同 | leader header-only body 重建错误 |
| leader 重建 hash 与 follower callback hash 不同 | DCF encode、传输、decode 或 buffer 生命周期问题 |
| DCF entry checksum 失败，原生 XLogReader 也在相同 record 失败 | 真实错误 WAL/body 的概率高 |
| DCF checksum 失败，但原生 XLogReader 正常 | DCF checksum 的范围、长度或表示语义不一致 |
| 只有发生 discontinuity reset 后才失败 | resync、continuation 或首条残缺 record 处理错误 |
| 失败 record 的 `page_remain` 总在 `[4, SizeOfXLogRecord)` | 固定 XLogRecord header 跨页拼接错误 |
| 只在 `entry_start/end` 跨 WAL segment 时失败 | header-only 跨 segment 文件读取错误 |
| stored CRC 固定，但 calculated CRC 每次 core 都不同 | 未初始化状态、未完全 reset 或 buffer 并发覆盖 |
| 备机 replay LSN 正常越过失败 record，redo 无原生 CRC 错误 | 自定义 checker 误报概率极高 |

## 最小定位顺序

按照下面顺序处理，可以减少一次性改动：

1. 比较连续两次 core 是否为同一 DCF index、record LSN 和 calculated CRC；
2. 确认备机 redo 是否能越过失败 record，或者使用原生 `XLogReader` 离线读取；
3. 增加 A/B/C 三处相同物理 LSN 区间的 hash；
4. 如果字节一致，检查 record 起点和 `page_remain`，优先排查固定 header 跨页；
5. 如果字节不一致，检查 header-only 回读的跨 segment、短读和 WAL write/flush 时序；
6. 第一轮证据不足时，再增加 `xlp_info/xlp_rem_len` 和逐 page hash，不要一开始打印所有 record 内容。

## 受控环境中放大复现概率

仅在测试环境中尝试：

- 缩小 DCF entry cache，增加 disk miss；
- 暂停一个 follower，再恢复使其批量追赶旧 index；
- 制造 ACK 丢失、重传、rematch 或 applied index rewind；
- 写入大量 WAL，并主动增加 WAL segment switch；
- 制造包含 FPI 的大 record，使其跨多个 page；
- 对比 header-only 优化开启和关闭；
- 分别统计失败 record 是否跨 page、segment，固定 header 是否跨 page。

目标不是构造某个业务 record 类型，而是放大物理边界。CRC 不解析 heap/btree 等 rmgr 业务含义；某类 record 更容易失败，通常是因为它更大、更容易跨 page/segment，或者触发 `XLOG_SWITCH`、continuation 和 padding 等特殊布局。

## 源码坐标

### openGauss

- `src/gausskernel/storage/access/transam/xlog.cpp`
  - `CopyXLogRecordToWAL()`：固定 header 可以跨页，起始页只保证容纳 `xl_tot_len`；
  - `XLogWritePaxos()`：DCF entry 的 `buf/nBytes/end LSN` 来源；
  - `XLogSelfFlushWithoutStatus()`：大 record 写满 page/WAL cache 环回时，以物理 page boundary 推进 Paxos write；
  - DCF 写入与本地 WAL write/flush 顺序。
- `src/gausskernel/storage/access/transam/xloginsert.cpp`
  - `XLogRecordAssemble()`：形成 `xl_tot_len` 和初始 CRC。
- `src/gausskernel/storage/access/transam/xlogreader.cpp`
  - `ValidXLogRecordHeader()`：record header 合法性检查；
  - `ValidXLogRecord()`：原生 CRC 计算和最终判定。
- `src/include/access/xlog_basic.h`
  - `XLogRecord`；
  - `SizeOfXLogRecord`；
  - `XLogRecordMaxSize`；
  - `XLogPageHeaderData`。
- `src/gausskernel/storage/replication/dcf/dcf_callbackfuncs.cpp`
  - `ReceiveLogCbFunc()`；
  - `XLogWalRcvReceive()` 前后的 index/LSN 推进。
- `src/gausskernel/storage/replication/walrcvwriter.cpp`
  - `walRcvWrite()`；
  - `XLogWalRcvWrite()`。

### DCF

- `src/replication/rep_leader.c`
  - `rep_appendlog_node()`：按 follower/index 获取并编码 entry；
  - `NEXT_INDEX` 前进、回退和重传。
- `src/replication/rep_common.c`
  - `rep_apply_proc()`：回调成功后才推进 applied index。
- `src/storage/stream.c`
  - `stream_append_entry()`；
  - `stream_get_entry()`；
  - entry cache 回收。
- `src/storage/log_storage.c`
  - `storage_get_entry()`：cache miss 后从 segment 读取。
- `src/storage/segment.c`
  - entry header/data checksum 的读取和验证。
- `src/replication/rep_msg_pack.c`
  - `rep_encode_one_log()`；
  - `rep_decode_one_log()`。

## 尚未确认的问题

- 私有 header-only 优化中，原始 WAL 长度和 DCF 实际落盘长度分别保存在哪个字段；
- 私有 checksum 的两次计算是否覆盖相同的字节范围；
- 出错 record 是否稳定满足固定 `XLogRecord` header 跨页条件；
- 两个备机失败时的 `record_start/xl_tot_len/calculated CRC` 是否完全相同；
- 第一次 core 前的 DCF index 是否已经写入备机 WAL 文件，以及重启后是重写还是只重新校验；
- 原生 redo/XLogReader 是否可以通过同一个失败 record；
- CRC checker 在 discontinuity reset 后是否从 page 中间猜测 record 起点；
- checker 是否错误地用 `record_start_lsn + xl_tot_len` 计算跨页 record 的物理 end LSN。

## 相关笔记

- [DCF 模式下 XLog Entry 的切分、传输与落盘流程](dcf-xlog-transport-flow.md)
- [WAL 回放专题入口](../../postgres/replay/wal-replay-study.md)
- [主备 WAL 全链路流程图](../../postgres/replay/overview-flow.md)
- [Replay 源码地图](../../postgres/replay/source-map.md)
