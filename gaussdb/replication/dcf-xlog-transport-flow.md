# openGauss DCF 模式下 XLog Entry 的切分、传输与落盘流程

## 文档信息

| 字段 | 内容 |
| --- | --- |
| 主题 | PostgreSQL WAL / openGauss DCF 复制 |
| 类型 | 源码阅读 |
| 状态 | 可用 |
| 创建时间 | 2026 年 8 月 22 日 |
| 更新时间 | 2026 年 8 月 23 日 |

## 要回答的问题

1. openGauss 在 DCF 模式下，WAL 是在哪里按约 1 MiB 切分的？
2. 一次 `dcf_write()`、一个 DCF entry、一个 DCF index 和一条 PostgreSQL WAL record 分别是什么关系？
3. DCF 网络层是否会再次拆分 entry，follower 又在哪里重新拼接？
4. `ReceiveLogCbFunc()` 和 `walreceiverwriter` 最终看到的 `buf/nbytes` 是否还保持 DCF entry 边界？
5. 这些边界对 WAL record CRC 校验有什么影响？

## 结论

- 约 1 MiB 的切分发生在 openGauss 的 `XLogWritePaxos()`，不是 DCF 库自动把一个 entry 切成多个 index。
- 一次 `dcf_write()` 对应一个 DCF entry 和一个新 index。标准开源实现会把这次调用传入的完整 payload 保存到 entry 中。
- 一个 DCF entry 是一段连续的物理 WAL 字节，可能包含多条完整 WAL record，也可能以某条 record 的 body 开始或结束。
- 达到 1 MiB 上限时，`XLogWritePaxos()` 会把 entry 末尾调整到 8 KiB WAL page 边界。因此它可以截断 WAL record，但标准路径下不会因为 1 MiB 限流而在物理 page 中间截断。
- DCF replication 层可以把多个 entry 编码进一个 AppendLog RPC；MEC 网络层还可以把过大的 RPC 拆成多个 fragment。follower 会先重组完整 RPC，再逐个解码和保存 entry。
- DCF apply 回调按 index 逐条调用，`ReceiveLogCbFunc()` 得到的是一个完整 entry 的 payload，而不是单个网络 fragment，也不是多个 entry 合并后的 payload。
- `ReceiveLogCbFunc()` 把 payload 写入 walreceiver 环形缓冲后，`walRcvWrite()` 只按“当前连续可读区域”取数据。它可能合并多个 DCF entry，也可能在环形缓冲回卷处拆开一个 entry。因此 walreceiverwriter 的 `nbytes` 没有 entry 或 WAL record 语义。
- `xl_tot_len` 是一条逻辑 WAL record 的总长度，不是 DCF entry 长度，也不是 walreceiverwriter 的 `nbytes`。
- CRC 校验必须把输入视为带物理 page header 的连续 WAL 字节流：跨调用保存 record 中间态、跳过 short/long page header 和对齐 padding，并且不能把一个 DCF index 当作一条 WAL record。

## 三类对象与四种长度

### PostgreSQL WAL record

一条逻辑 WAL record 由固定 `XLogRecord` header 和后续 record data 构成：

```text
XLogRecord header
  - xl_tot_len
  - xl_term
  - xl_xid
  - xl_prev
  - xl_info
  - xl_rmid
  - xl_bucket_id
  - xl_crc
+ block/data headers
+ FPI / block data / main data
```

`xl_tot_len` 表示这一整条逻辑 record 的长度：

- 包含固定 `XLogRecord` header；
- 包含 block/data header、FPI 和 main data；
- 不包含物理 WAL page header；
- 不包含 record 末尾为了下一条 record 对齐而增加的 padding。

因此一条 WAL record 超过 10 KiB 是正常的。包含多个 full-page image 时，一条 record 可以明显大于 8 KiB，源码允许的最大 record 长度接近 1 GiB。

### 物理 WAL page

物理 WAL 按 `XLOG_BLCKSZ`，通常为 8 KiB，组织成 page：

```text
普通 page:
  short page header
  record / record continuation bytes

segment 第一个 page:
  long page header
  record / record continuation bytes
```

record 跨 page 时，下一个 page 会设置：

```text
XLP_FIRST_IS_CONTRECORD
xlp_rem_len
```

page header 后首先出现的是上一条 record 的 continuation body，不是新的 `XLogRecord` header，也没有新的 `xl_tot_len`。

### DCF entry/index

一个 DCF entry 是一次 `dcf_write()` 写入的 payload，并由 DCF 分配一个 index。它只是 WAL 物理字节流的一个传输和一致性提交单元，不是 PostgreSQL WAL record。

### 长度对照

| 长度 | 所属层 | 含义 | 是否与 WAL record 对齐 |
| --- | --- | --- | --- |
| `xl_tot_len` | WAL record | 一条逻辑 WAL record 的总长度 | 本身就是 record 长度 |
| `XLOG_BLCKSZ` | WAL 物理页 | short/long page header 加本页 WAL 数据 | 8 KiB page 边界，不是 record 边界 |
| `nBytes` / DCF entry size | openGauss → DCF | 一次 `dcf_write()` 的 WAL 片段长度，最大约 1 MiB | 可以在 record body 中截断 |
| `nbytes` in `walRcvWrite()` | walreceiver ring | 当前环形缓冲中连续可写盘的数据长度 | 可能合并或拆分 entry，也不要求 record 对齐 |

## 总体调用链

```text
XLogInsertRecord
  │ 生成逻辑 WAL record，写入 WAL buffer
  │ xl_tot_len / xl_crc 在这里形成
  ▼
XLogBackgroundFlush / XLogSelfFlushWithoutStatus
  ▼
XLogWritePaxos(PaxosWriteRqst)
  │ 从 WAL buffer 中取得连续物理 WAL 字节
  │ 按 WriteTotal、1 MiB、WAL cache 连续空间选择 nBytes
  ▼
dcf_write(stream=1, from, nBytes, WriteLSN, &index)
  │ 一次调用创建一个 DCF entry/index
  │ key = WriteLSN = entry end LSN
  ▼
DCF leader storage/cache
  ▼
rep_appendlog_node
  │ 一个 AppendLog RPC 可以编码一个或多个 entry
  ▼
MEC transport
  │ RPC 过大时拆成 fragment
  ▼
follower concat_fragment_pack
  │ 先重组完整 RPC
  ▼
rep_decode_one_log → stg_append_entry
  │ 按 index 恢复并保存每个完整 entry
  ▼
rep_apply_proc
  │ 按 applied_index + 1 顺序逐个 apply
  ▼
ReceiveLogCbFunc(index, ENTRY_BUF, ENTRY_SIZE, key)
  │ start LSN = key - size
  │ 可能根据 segment 对齐要求裁掉前缀
  ▼
XLogWalRcvReceive(copyStart, receive_len, copyStartPtr)
  │ 复制进 walreceiver 环形缓冲
  ▼
walRcvWrite
  │ 取当前连续可读区域
  ▼
XLogWalRcvWrite
  │ 写入备机 WAL 文件
  ▼
startup / redo
```

## 1. 主机 WAL record 的形成

WAL insert 端先生成 record data，再形成固定 `XLogRecord` header。

CRC 使用 CRC32C，计算范围包括：

- record data；
- 固定 record header 中 `xl_crc` 之前的字段。

不包括：

- `xl_crc` 字段自身；
- short/long WAL page header；
- record 末尾的 `MAXALIGN` padding。

源码中的 CRC 顺序是先计算 record data，再计算 `XLogRecord` header 中 `xl_crc` 之前的内容。这与物理 WAL 中 header 先出现的顺序不同，增量校验器需要保留完整 record header，最后按原生顺序完成 CRC。

## 2. `XLogWritePaxos()` 如何生成约 1 MiB 的 entry

本地 openGauss 定义：

```cpp
static const Size MaxSendSizeBytes = 1048576;
```

每轮计算：

```cpp
WriteTotal = PaxosWriteRqst - LogwrtPaxos->Write;

CriticalNBytes =
    WAL cache 从当前位置到数组末尾的连续字节数;

nBytes = Min(WriteTotal,
             Min(MaxSendSizeBytes, CriticalNBytes));
```

三种限制分别表示：

1. `WriteTotal`：这次最多只能推进到请求的 Paxos write LSN；
2. `MaxSendSizeBytes`：单次发送最多约 1 MiB；
3. `CriticalNBytes`：不能从 WAL 环形 cache 的物理尾部一次连续取到头部。

如果真正命中的是 1 MiB 上限，代码继续调整：

```cpp
tmpLsn = currentWriteLsn + MaxSendSizeBytes;
tmpoffset = tmpLsn % XLOG_BLCKSZ;
nBytes = MaxSendSizeBytes - tmpoffset;
```

调整后的 entry end LSN 位于下一个 WAL page 边界。

例如当前 LSN 的 page offset 为 100：

```text
初始限制: 1,048,576 bytes
调整后:   1,048,576 - 100 bytes
entry end: 8 KiB page boundary
```

后续如果从 page boundary 开始，常见 entry 长度就是完整的 1 MiB，因为 1 MiB 本身是 8 KiB 的整数倍。

### 对 record 边界的影响

这种切法只保证 page 对齐，不保证 record 对齐：

```text
DCF entry i:
  page header
  record A
  record B header + body 的前半部分
  <entry end，同时也是 page end>

DCF entry i+1:
  continuation page header
  record B body 的后半部分
  record C
```

所以：

- 一个 entry 可以包含多条 record；
- 一条 record 可以跨多个 entry；
- 下一个 entry 可以从 continuation page header 开始；
- DCF entry 的 `len` 不能拿来与某条 record 的 `xl_tot_len` 比较。

### `key/lsn` 的含义

openGauss 调用：

```cpp
WriteLSN = LogwrtPaxos->Write + nBytes;
dcf_write(1, from, nBytes, WriteLSN, &paxosIdx);
```

因此标准路径中：

```text
entry_end_lsn   = key = WriteLSN
entry_start_lsn = key - len
```

这里的 key 是 entry end LSN，而不是 record start LSN 或 record end LSN。

## 3. 一次 `dcf_write()` 如何成为一个 DCF entry

开源 DCF 中调用关系是：

```text
dcf_write
  → rep_write
    → stg_append_entry
      → stream_append_entry
```

`stream_append_entry()` 会：

1. 为传入的完整 `data/size` 分配 entry buffer；
2. leader 为它分配一个新 index；
3. 填充 entry header；
4. 把 entry 放入 cache；
5. 通知 disk thread。

因此标准实现中的关系是：

```text
一次 dcf_write
  = 一个 entry
  = 一个 index
  = 一段完整保存的 payload
```

DCF entry 自身有独立的存储 header，包括：

```text
term
index
key
entry type
payload size
payload checksum
entry header checksum
```

这些是 DCF 存储层元数据，不属于 PostgreSQL WAL 字节流。回调中的 `ENTRY_BUF(entry)` 已经跳过 DCF entry header。

## 4. leader 如何打包多个 entry

`rep_appendlog_node()` 从 `NEXT_INDEX` 开始依次读取 entry，并调用：

```cpp
rep_encode_one_log(..., entry);
```

一条编码后的 log 包含：

```text
term
index
payload size + payload
entry type
key
```

一个 AppendLog RPC 可以携带多个连续 index。replication 层使用约 64 KiB 的 `MESSAGE_BUFFER_SIZE` 控制一次通常打包多少数据。

如果第一个 entry 本身已经大于 64 KiB，`j == 0` 时仍然会把它完整放进当前 RPC。因此约 1 MiB 的 entry 不会被 replication 层改造成多个 DCF entry，只会让整个 RPC 进入后续的网络分片流程。

## 5. MEC 网络 fragment 与重新拼接

openGauss 默认设置：

```text
dcf_mec_fragment_size = 64 KiB
```

允许范围为 32 KiB 到 10 MiB。

当 RPC 大于 MEC message buffer 时，`mec_send_fragment()` 会设置：

```text
CS_FLAG_MORE_DATA
CS_FLAG_END_DATA
frag_no
```

并把 RPC 拆成多个网络 fragment。

follower 接收端的处理是：

```text
dtc_proc_more_data
  → 暂存或追加 fragment

dtc_proc_end_data
  → concat_fragment_pack
  → 构造完整 mec_message_t
  → 调用真正的 AppendLog RPC processor
```

因此网络 fragment 对上层 entry apply 回调不可见：

```text
1 MiB entry
  → 网络上可能是十几个 64 KiB fragment
  → follower 先拼回完整 RPC
  → 再解码得到原始 1 MiB entry
  → 回调一次
```

这里的“拼接”是网络 RPC 重组，不是 PostgreSQL WAL record 重组。

## 6. follower 如何恢复并保存 entry

follower 先解析 AppendLog RPC header，再根据 `log_count` 逐个执行：

```text
rep_decode_one_log
  → 得到 term/index/buf/size/type/key
  → stg_append_entry
```

即使一个 RPC 中带了多个 entry，follower 也会逐个恢复它们的 index、payload size 和 payload，再分别写入 DCF storage/cache。

所以不能把下面三者混为一谈：

```text
网络 fragment 边界
AppendLog RPC 边界
DCF entry/index 边界
```

## 7. committed entry 如何进入 openGauss 回调

`rep_apply_proc()` 从：

```text
applied_index + 1
```

开始，按 index 顺序处理到 `commit_index`。

对于每个 entry：

- leader/current term 调用 `after_writer`；
- follower 或其他情况调用 `consensus_notify`。

回调参数直接使用：

```cpp
ENTRY_BUF(entry)
ENTRY_SIZE(entry)
ENTRY_KEY(entry)
```

所以 `ReceiveLogCbFunc()` 看到的是单个、完整 entry。

### 回调失败与重试

如果回调返回非零，`rep_apply_proc()` 会在更新 applied index 之前返回。下一轮 apply 仍可能再次处理同一个 index。

因此在回调中维护跨 entry CRC 中间态时要特别注意：

- 不能先推进 CRC 状态，再通过后续检查返回失败；
- 否则同一个 entry 重试时会被累计两遍；
- CRC feed 应放在所有可能提前失败的检查之后；
- 或者让 CRC 状态更新具备事务性，回调成功时才提交中间态；
- 至少保存最后已消费的 index 和 `[start_lsn, end_lsn)`，识别重复回调。

## 8. `ReceiveLogCbFunc()` 对 payload 的处理

openGauss follower 回调首先计算：

```cpp
paxosStartPtr = lsn - len;
paxosEndPtr   = lsn;
```

然后根据当前 receive segment 做一次可能的前缀裁剪：

```cpp
copyStart    = buf + alignOffset;
receive_len  = len - alignOffset;
copyStartPtr = paxosStartPtr + alignOffset;
```

真正送入 walreceiver 的是：

```cpp
XLogWalRcvReceive(copyStart, receive_len, copyStartPtr);
```

因此如果 CRC 校验放在这个位置，应校验实际消费的：

```text
[copyStartPtr, copyStartPtr + receive_len)
```

而不是无条件校验原始的：

```text
[paxosStartPtr, paxosStartPtr + len)
```

## 9. walreceiver 环形缓冲如何再次改变边界

`XLogWalRcvReceive()` 把 entry payload 复制到 SPSC 风格的 walreceiver 环形缓冲。一次 entry 可能因为以下原因被分段复制：

- 当前环形缓冲尾部空间不足；
- producer 需要等待 walreceiverwriter 释放空间；
- buffer 到达数组末尾并回卷到 offset 0。

`walRcvWrite()` 并不知道 DCF index。它只观察：

```text
walFreeOffset
walWriteOffset
walStart
```

并计算当前内存中连续可读的 `nbytes`：

```cpp
if (walFreeOffset < walWriteOffset)
    nbytes = recBufferSize - walWriteOffset;
else
    nbytes = walFreeOffset - walWriteOffset;
```

这会产生两种变化。

### 合并

如果 producer 连续写入多个 entry，writer 一次被唤醒时可能看到：

```text
entry 100 + entry 101 + entry 102 的部分或全部
```

于是 `walRcvWrite()` 的一个 `nbytes` 覆盖多个 DCF index。

### 拆分

如果某个 entry 跨过环形缓冲数组末尾：

```text
walRcvWrite #1: entry 前半段，写到 ring end
walRcvWrite #2: entry 后半段，从 ring offset 0 开始
```

这个 ring 边界不携带 WAL record 语义，因此理论上可以落在：

- WAL page header 中间；
- `xl_tot_len` 的四字节中间；
- 固定 `XLogRecord` header 中间；
- record body 中间；
- record 对齐 padding 中间。

所以即使原始 DCF entry 边界经过了 WAL page 对齐，walreceiverwriter 的 `buf/nbytes` 也不能沿用这个假设。

## 10. 对 CRC 校验器的直接影响

### 不能依赖的假设

以下假设都不安全：

```text
一次 DCF 回调就是一条 WAL record
一次 walRcvWrite 就是一个 DCF entry
buf 的开头一定是 XLogRecord header
buf 的末尾最多只残留 record header
只要遇到 record 开头，当前 buf 中一定有完整 xl_tot_len
一次输入中出现的第一个不完整片段一定能直接跳过
```

### 标准 DCF entry 边界上的特殊结论

标准开源路径中，命中 1 MiB 限流时，entry end 会对齐到 WAL page boundary。结合 WAL insert 保证 record 开始页至少能容纳 `xl_tot_len` 的四字节，可以推导：

```text
1 MiB DCF entry 边界可以截断 record body，
但通常不会把 xl_tot_len 四字节切成 1+3、2+2 或 3+1。
```

这只是对 `XLogWritePaxos()` 原始 entry 边界的结论，不适用于 walreceiver 环形缓冲边界，也不适用于修改过 entry 组装方式的私有实现。

### CRC 校验状态至少需要表达

```text
expected physical WAL LSN
当前是否正在收集 short/long page header
已保存的 page header 字节数
是否处于 continuation record
当前 record start LSN
已保存的 xl_tot_len 字节数，范围 0..4
已保存的固定 XLogRecord header 字节数
当前 record 的 xl_tot_len
当前 record 已累计的逻辑字节数
CRC32C 中间态
当前是否在跳过 MAXALIGN padding
最后成功消费的 DCF index（如果校验位于回调中）
```

### 输入连续性

设本次有效输入范围为：

```text
[start_lsn, end_lsn)
```

应当处理：

```text
start_lsn == expected_lsn
    正常继续

start_lsn > expected_lsn
    出现 gap，当前 record 状态失效，重新同步

start_lsn < expected_lsn
    可能是重复回调、重试或重叠数据，不能直接重复累计 CRC
```

### 从任意 LSN 重启时的同步限制

如果 CRC 状态被清空，而新输入从一个 WAL page 中间开始，仅凭当前位置的字节无法可靠区分：

- `XLogRecord` header；
- record body；
- record alignment padding。

安全做法是放弃当前 page 的剩余部分，从下一个可信 page header 开始同步：

1. 读取并验证 short/long page header；
2. 如果设置了 `XLP_FIRST_IS_CONTRECORD`，根据 `xlp_rem_len` 跳过前一条 record 的 continuation；
3. 到达第一个可以确定的新 record 起点后再开始 CRC 校验。

这样可能少校验一条跨越重启点的 record，但不会因为猜错 record 边界而误报。

## 11. 建议的观测日志

本节只列传输链路上的基础观测点。针对偶现 CRC 失败、重复 core、header-only body 重建、core/GDB 取证和最小诊断循环缓冲的完整操作步骤，参见 [DCF 模式下 XLog CRC 偶现失败调试手册](dcf-xlog-crc-debugging.md)。

### 在 `XLogWritePaxos()` 调用 `dcf_write()` 前

```text
index
start_lsn
end_lsn
nBytes
start_lsn % XLOG_BLCKSZ
end_lsn % XLOG_BLCKSZ
WriteTotal
CriticalNBytes
是否命中 MaxSendSizeBytes
物理 payload hash
```

### 在 `ReceiveLogCbFunc()`

```text
index
paxosStartPtr
paxosEndPtr
len
alignOffset
copyStartPtr
receive_len
物理 payload hash
```

### 在 `walRcvWrite()`

```text
startptr
startptr + nbytes
nbytes
walWriteOffset
walFreeOffset
是否在本次写入后发生 ring wrap
物理 payload hash
```

### CRC 首次失败时

```text
parser state
expected_lsn
input start/end LSN
record_start_lsn
xl_tot_len saved bytes
record header saved bytes
record logical bytes collected
expected xl_crc
calculated crc
last DCF index
```

按相同 `[start_lsn, end_lsn)` 在主机 DCF 写入前、follower callback 和 walreceiverwriter 计算一个简单物理字节 hash，可以先判断：

- hash 已经不同：entry 重建、磁盘 body 读取或传输链路有问题；
- hash 相同但 record CRC 不同：自定义 WAL record 解析器或 CRC 状态机有问题。

## 12. 当前源码未覆盖的私有优化

当前检查的源码版本：

- 本地 openGauss：`dev_storage`，commit `33f3e1485758e2d27287b608f89f5e2515451c30`；
- GitHub DCF：`master`，检查时 commit `ab2aa80b0ec3a558d4fc893cde81e3f878501556`。

这两个版本中，DCF storage 保存的是完整 entry payload。没有找到下面这种私有优化：

```text
leader 在 DCF 中只保存引用 header
  → 发送时根据 header 从本地 WAL 文件读取 body
  → header + body 重新组装后发送 follower
```

如果实际运行分支包含这一优化，还必须额外确认：

1. 回调看到的是引用 header，还是重新组装后的完整 WAL payload；
2. 组装后的 `len/key` 是否仍满足 `start_lsn = key - len`；
3. 从本地 WAL 文件读取 body 时，所需范围是否已经完成 local write/flush；
4. 是否正确处理短读、WAL segment 边界、timeline 和 segment recycle；
5. entry payload 在组装前后是否保持逐字节一致。

在拿到实际私有分支或 diff 后，需要把这段优化补到本文的 entry storage 和 leader send 两节之间。

## 源码坐标

### 本地 openGauss

- `src/gausskernel/storage/access/transam/xlog.cpp`
  - `MaxSendSizeBytes`
  - `XLogWritePaxos()`
  - `dcf_write(1, from, nBytes, WriteLSN, &paxosIdx)`
- `src/gausskernel/storage/replication/dcf/dcf_callbackfuncs.cpp`
  - `ReceiveLogCbFunc()`
  - `ConsensusLogCbFunc()`
- `src/gausskernel/storage/replication/dcf/dcf_replication.cpp`
  - DCF callback 注册
  - `MEC_FRAGMENT_SIZE` 和 `MEC_BATCH_SIZE` 参数传递
- `src/gausskernel/storage/replication/walreceiver.cpp`
  - `XLogWalRcvReceive()`
- `src/gausskernel/storage/replication/walrcvwriter.cpp`
  - `walRcvWrite()`
  - `XLogWalRcvWrite()`
- `src/gausskernel/storage/access/transam/xloginsert.cpp`
  - record data CRC 的初始计算
- `src/gausskernel/storage/access/transam/xlogreader.cpp`
  - 跨 page record 重组
  - `ValidXLogRecord()`
- `src/include/access/xlog_basic.h`
  - `XLogRecord`
  - `XLogPageHeaderData`
  - `SizeOfXLogRecord`

### GitHub DCF

- [`src/dcf_interface.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/dcf_interface.c#L875-L903)
  - `dcf_write()`
- [`src/storage/stream.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/storage/stream.c#L423-L465)
  - `stream_append_entry()`
- [`src/replication/rep_leader.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/replication/rep_leader.c#L397-L445)
  - `rep_appendlog_node()`
- [`src/replication/rep_msg_pack.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/replication/rep_msg_pack.c#L65-L90)
  - `rep_encode_one_log()`
  - `rep_decode_one_log()`
- [`src/replication/rep_follower.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/replication/rep_follower.c)
  - `rep_appendlog_req_proc()`
  - `rep_follower_appendlog()`
- [`src/replication/rep_common.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/replication/rep_common.c#L197-L245)
  - `rep_apply_proc()`
- [`src/network/mec/mec_func.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/network/mec/mec_func.c#L1938-L2010)
  - `mec_send_fragment()`
- [`src/network/mec/mec_queue.c`](https://github.com/opengauss-mirror/DCF/blob/ab2aa80b0ec3a558d4fc893cde81e3f878501556/src/network/mec/mec_queue.c#L687-L795)
  - `dtc_proc_more_data()`
  - `dtc_proc_end_data()`

## 还没搞懂

- 实际运行分支中的 header-only DCF entry 优化位于哪些函数，entry storage 和 replication message 格式分别发生了什么变化。
- 当前误校验第一次出现时，错误 record 的输入边界来自 DCF index、`alignOffset`、walreceiver ring wrap，还是 callback retry。
- 稳定的流复制 CRC 实现是否隐含依赖了 walsender 起始 LSN、发送粒度或首次输入必然位于可信 record/page 边界。
- walreceiverwriter 中 CRC 状态是否会被 DCF callback 和流复制路径重复使用，或被重复 entry 再次推进。

## 相关笔记

- [DCF 模式下 XLog CRC 偶现失败调试手册](dcf-xlog-crc-debugging.md)
- [WAL 回放专题入口](../../postgres/replay/wal-replay-study.md)
- [主备 WAL 全链路流程图](../../postgres/replay/overview-flow.md)
- [WAL insert head/tail 与写入推进](../../postgres/replay/concepts/insert-pointers.md)
- [Replay 源码地图](../../postgres/replay/source-map.md)
