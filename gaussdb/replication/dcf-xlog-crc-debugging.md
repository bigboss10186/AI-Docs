# openGauss DCF 模式下 XLog CRC 偶现失败调试手册

## 文档信息

| 字段 | 内容 |
| --- | --- |
| 主题 | openGauss DCF / WAL record CRC / core 定位 |
| 类型 | 排障复盘 |
| 状态 | 可用 |
| 创建时间 | 2026 年 8 月 23 日 |
| 更新时间 | 2026 年 8 月 26 日 |

## 目标与范围

本文整理 DCF 模式下复用 `check_xlog_buf()` 做 WAL record CRC 校验时，接收端和发送端分别需要满足的边界条件，并记录一次接收端误校验的现有证据、最可能根因和后续取证方法。

当前实现包含两个校验位置：

- 接收端：`walreceiverwriter` 从 walreceiver 环形缓冲取出 `buf/nbytes/startPtr`，写盘前调用 `check_xlog_buf()`；
- 发送端：DCF leader 在 `rep_appendlog_node()` 中逐 index 调用 `stg_get_entry()`，完成私有 header-only WAL body 拼装后、`rep_encode_one_log()` 前调用相同的 `check_xlog_buf()`。

发送端的目标是 best-effort：只校验从磁盘读取且能够可靠恢复 record 边界的内容，允许漏校验，但不能因为 entry 不连续而误报 CRC 失败。

DCF entry 的完整切分、网络重组和 walreceiver 缓冲流程见 [DCF 模式下 XLog Entry 的切分、传输与落盘流程](dcf-xlog-transport-flow.md)。

## 当前结论摘要

### 接收端

- 当前 core 已经证明：checker 把真实 WAL record body 中的 `E3/B90D4698` 当成了下一条 `WalRecord` header。
- 从该位置读出的 `xl_tot_len = 0x3c989677 = 1016632951` 是 fake header 字段，不是真实 record 长度。
- `ready_data_len` 最终也达到 `1016632951`，说明状态机确实从错误 record 边界持续累计了约 1 GiB，最后才执行 CRC 并 PANIC。
- 测试会 kill 备机。重启后 CRC 线程状态全部初始化，同时 DCF applied/index 发生回退，主机重发一段旧 WAL。重发字节可以完全正确，但第一次校验起点未必是 WAL record 起点。
- 非 TDE 路径原本依靠 `is_first_check`：第一次从非 record 边界解析得到 CRC mismatch 后跳到下一 WAL page，再通过 page header 和 `XLP_FIRST_IS_CONTRECORD` 找回可信 record 边界。
- 当前 TDE 分支会在 fake header 恰好满足 `IS_RECORD_ENCRYPTED()` 时跳过 CRC mismatch；`latest_end_ptr` 已经提前更新且没有回退，`is_first_check` 又被清除，于是错误边界继续传播。
- 直接故障原因已经确定；“首次 fake encrypted record 绕过恢复逻辑”是当前概率最高的上游原因，但仍需捕获第一次污染现场才能达到完全闭环。

### 发送端

- 一个 DCF entry 是一段连续物理 WAL，不是一条 WAL record。一个 entry 可以包含多条 record，一条 record 也可以跨多个 entry。
- 单个 entry 不超过约 1 MiB 不构成额外适配要求；`check_xlog_buf()`本身能够跨调用保存 fixed header、record body、page header 和 CRC 中间态。
- 同一 node 的连续磁盘 entry 可以跨 entry 校验；中间存在 cache hit、回退、重传或 index 跳变时，`startPtr != check_end_ptr`，checker 应完整 clean 后重新同步。
- 发送端只校验磁盘 entry 会降低覆盖率，但符合当前目标。未能确认 record 边界时应跳过，不能报 CRC 失败。
- 接收端发现的 `is_first_check`/TDE 缺陷同样影响发送端。统一修复后，不需要为 1 MiB entry 重新实现一套 WAL parser。

## 三种边界必须分开

| 边界 | 含义 | 是否保证是 record 边界 |
| --- | --- | --- |
| WAL record boundary | 一条逻辑 record 的起止位置 | 是 |
| WAL page boundary | 8 KiB 物理 page 起点，先出现 short/long page header | 否；page 后可能是上一条 record continuation |
| DCF entry/index boundary | 一次 `dcf_write()` 的连续 WAL payload | 否；可以包含多条 record 或截断 record body |
| walreceiverwriter `buf/nbytes` boundary | 环形缓冲当前连续可读区间 | 否；可能合并或拆开 DCF entry |

`check_end_ptr == startPtr` 只证明两次输入的物理 LSN 连续，不证明 `startPtr` 是 record boundary。

例如：

```text
entry N:
  record A 前半部分
  check_end_ptr = X

entry N+1:
  startPtr = X
  record A 后半部分
```

此时 `startPtr == check_end_ptr` 完全正常。上一轮应保留 `is_remaining_xlog=true`、`latest_wal_record` 和 `ready_data_len`，下一轮继续走 `check_from_middle()`。

真正异常的组合是：

```text
startPtr == check_end_ptr
is_remaining_xlog == false
startPtr 又位于 record body
```

在状态机正确的前提下，这种组合不应自然出现；它通常意味着状态已经被错误推进、保存/恢复不完整，或者相同 LSN 的输入语义发生了变化。

## `check_xlog_buf()` 的状态模型

至少需要同时保存：

```text
latest_comp_crc
latest_check_ptr
latest_end_ptr
check_end_ptr
latest_wal_record_valid_size
latest_wal_record
is_remaining_xlog
ready_data_len
is_remain_page_header
page_header_info
page_header_rem_len
is_first_check
```

连续输入时，这些字段共同描述当前 record/header/page continuation 的解析进度。发生 gap、overlap、rewind 或 checker 初始化时，不能只清一两个长度字段，必须把整组状态恢复到一致的初始值。

初始化或 clean 后的关键状态是：

```text
check_end_ptr     = INVALID
latest_end_ptr    = INVALID
is_remaining_xlog = false
is_first_check    = true
ready_data_len    = 0
header/CRC buffer = empty
```

如果 `check_xlog_buf()` 入口发现：

```cpp
startPtr != check_end_ptr
```

当前实现会 clean，然后继续处理本次 `buf`。如果 `startPtr` 位于 page 中间，`check_from_start()` 无法仅凭字节内容判断当前位置是 record header、record body 还是 padding，只能依靠首次失败后的 resync 机制恢复。

## 接收端故障证据

### 真实 WAL 布局

原生 XLog 信息为：

```text
REDO @ E3/B90D31E0
LSN  E3/B90D4EB0
total 7350
UHeap - uheap_new_page
this xlog has been encrypted
```

物理布局可以还原为：

```text
E3/B90D31E0  record start
      |
      | 3616 bytes record data
      v
E3/B90D4000  WAL page boundary
      |
      | 24 bytes short page header
      v
E3/B90D4018  record continuation
      |
      | record 后半部分
      v
E3/B90D4EB0  aligned next position
```

`E3/B90D4698` 位于 `[E3/B90D4018, E3/B90D4EB0)` 内，距离 continuation 起点 `0x680 = 1664` 字节，因此绝不是 record boundary。

### Core 与 PANIC

PANIC 位置：

```text
incorrect wal record in E3/B90D4698
wal total len 1016632951
```

core 中保存的 fake header：

```text
xl_tot_len = 0x3c989677 = 1016632951
ready_data_len = 1016632951
latest_check_ptr = E3/B90D4698
latest_end_ptr ≈ E3/F5D37568
is_remaining_xlog = true
is_first_check = false
```

这组值互相吻合：checker 从 `E3/B90D4698` 的普通 payload 字节中读出约 1 GiB 的 fake `xl_tot_len`，随后跨大量 page 和输入批次累计数据；物理 page header 不计入 `ready_data_len`，因此最终物理结束位置还包含额外 page-header 开销。

当前 core 是第二条 fake record 的最终现场，无法恢复更早第一次把 `latest_check_ptr` 推到 `E3/B90D4698` 时的 header 字节。

## 接收端最可能触发链

测试过程提供了重要前置条件：

```text
kill 备机
  → walreceiverwriter/CRC TLS 状态全部重置
  → 备机重启
  → DCF index/applied 状态回退
  → 主机重发一段旧 WAL
```

第一次进入 checker 时 `check_end_ptr=INVALID`，因此不会产生 `startPtr != check_end_ptr` 的 clean 日志，而是直接使用 `is_first_check=true` 处理重发起点。

当前最可能存在两条 fake record：

```text
fake record A
  重发起点不在 record boundary
    ↓
  body/padding 被解释成 WalRecord header
    ↓
  fake header 恰好满足 IS_RECORD_ENCRYPTED
    ↓
  CRC mismatch 被 TDE 条件静默跳过
    ↓
  latest_end_ptr 已更新且不回退
    ↓
  is_first_check 被置为 false
    ↓
  下一位置被推进到 E3/B90D4698

fake record B
  从 E3/B90D4698 读取 header
    ↓
  xl_tot_len = 0x3c989677
    ↓
  累计约 1 GiB
    ↓
  当前 fake header 未命中 encrypted
    ↓
  CRC mismatch + is_first_check=false
    ↓
  PANIC
```

这可以同时解释：

- 为什么问题偶现：非对齐字节还要恰好命中 fake encrypted 条件；
- 为什么第一次 core 后可能持续 core：每次重启都从相同 DCF entry/LSN 和相同字节重新开始，判断结果是确定性的；
- 为什么两个备机可能在同一个位置 core：两边经历相同的 reset、回退和重发起点；
- 为什么原生 WAL record 看起来正常：真正错误的是 checker 识别出的 record boundary，而不是必然存在物理 WAL 损坏。

## 接收端修复原则

### 首次恢复优先于 TDE 例外

当前危险逻辑等价于：

```text
encrypted && CRC match      → 特殊处理
!encrypted && CRC mismatch  → first-check resync / PANIC
encrypted && CRC mismatch   → 静默成功
```

首次或失步状态下，`WalRecord` header 尚未可信，不能先信任从该 header 读取的 `IS_RECORD_ENCRYPTED()`。

正确优先级应为：

```text
CRC mismatch
  if is_first_check / unsynced
      无论 encrypted 位是什么，都执行 page resync
  else if 已确认是真实 encrypted record
      使用 TDE 的特殊语义
  else
      retry / PANIC
```

修复时还应满足：

1. CRC 和结构校验成功前，只计算 `candidate_end_ptr`，不要提前提交 `latest_end_ptr`；
2. first-check 失败后清理 fake record 的 header、`ready_data_len` 和 CRC 中间态；
3. `is_first_check` 保持 true，直到 page/CONTRECORD resync 完成或第一条可信 record 验证成功；
4. 跳到下一物理 page 后先处理 short/long page header；
5. `XLP_FIRST_IS_CONTRECORD` 置位时按 `xlp_rem_len` 跳过 continuation 和 alignment padding，再开始下一条 record。

## 发送端接入结论

### 调用位置

发送端回调位于 `rep_appendlog_node()` 的 index 循环内：

```text
stg_get_entry(stream_id, index)
  → 私有版本完成 header-only WAL body 拼装
  → 对 ENTRY_TYPE_LOG 调用内核 check_xlog_buf()
  → rep_encode_one_log()
  → mec_send_data()
```

一次 AppendLog RPC 可以包含多个 entry，但回调仍按 entry 逐个执行。回调参数必须与最终发送数据一致：

```text
buf       = assembled ENTRY_BUF(entry)
len       = assembled/original WAL length
end_lsn   = ENTRY_KEY(entry)
start_lsn = end_lsn - len
```

不能用 header-only 引用 header 的物理存储长度计算 `start_lsn`。

### 线程和 node 状态

开源 DCF 使用 `g_append_thread_id[node_id]` 将同一 follower 固定到一个 append 线程。一个线程仍可能轮询多个 node，因此私有内核回调需要按目标 `node_id` 保存/切换完整 CRC 状态槽。

切换时必须保存前述全部状态，包括：

```text
latest_wal_record 字节
latest_wal_record_valid_size
ready_data_len
page-header partial state
is_remaining_xlog
is_first_check
check_end_ptr/latest_* ptr
CRC 中间态
```

只保存 LSN 和 CRC 数值、不保存 partial header/body，会在 node 切回时制造新的 fake record。

### 只校验磁盘 entry

当前目标允许跳过 cache hit entry，因此输入分为：

```text
连续磁盘 entry
  → startPtr == check_end_ptr
  → 保留中间态，允许一条 record 跨多个 entry

中间存在 cache hit entry
  → 下一个磁盘 entry 的 startPtr != check_end_ptr
  → clean
  → is_first_check=true
  → 从当前输入重新同步
  → 中间 record 可以漏检，但不能误报
```

一个 entry 小于约 1 MiB 不影响 parser 正确性：

- 一个 entry 可以包含多条完整 record；
- entry 可以以某条 record body 开始或结束；
- 一条大 record 可以跨多个 entry；
- fixed header 或 page header 也可以跨 checker 调用。

因此，统一修复 `is_first_check`/TDE 优先级后，`check_xlog_buf()`可以直接复用于发送端，无需按 1 MiB entry 重新实现 parser。代价只是磁盘 entry 之间存在空洞时覆盖率下降。

### 发送端 best-effort 判错条件

只有下面条件全部成立时，发送端才应该上报 CRC mismatch：

```text
1. 当前已经找到可信 record boundary；
2. header 已完整拼接并通过基本结构检查；
3. 本条 record 的字节来自连续的已校验 entry；
4. record body 已完整收齐；
5. 当前不是 unsynced/first-check 恢复阶段；
6. TDE record 使用了正确的可校验语义；
7. stored CRC 与 calculated CRC 仍不一致。
```

任何条件不满足时，应丢弃未完成状态并重新同步，而不是 PANIC。这个策略符合“允许漏校验，但不允许误校验失败”的目标。

## `wal_total_len` 为 0 或过小

合法 record 的 `wal_total_len/xl_tot_len` 不可能小于对应 fixed header。下面的写法存在无符号下溢风险：

```cpp
wal_total_len - SIZE_OF_WAL_RECORD
```

当 `wal_total_len=0` 时，结果会变成一个很大的无符号数，随后 CRC、拷贝或长度累计可能越界并 core。ARM 可能更容易暴露，但根因与架构无关。

必须在任何减法和 CRC feed 之前检查：

```cpp
if (!header_complete) {
    /* 继续收集header，不能读取长度之外的字段 */
}

uint32 header_size = is_new_record
    ? SIZE_OF_WAL_RECORD(WalRecord)
    : SIZE_OF_WAL_RECORD(OWalRecord);

if (wal_total_len < header_size ||
    wal_total_len > WAL_RECORD_MAX_SIZE) {
    /* 禁止继续减法、拷贝和CRC */
}
```

处理方式按同步状态区分：

```text
is_first_check=true / unsynced
  → 认为当前 header 不可信
  → 清理当前 fake record
  → 跳到下一 WAL page
  → 不报 CRC 错误

已经处于可信 record 流
  → 可能是真实 WAL/header 损坏
  → 重新读取确认，或按严格策略报错
```

发送端 best-effort 模式可以选择 reset + 跳页。判断非法后不能只增加 `offset`，必须同步清理 `latest_wal_record_valid_size`、`ready_data_len`、`is_remaining_xlog` 和 CRC 中间态。

仅增加上限检查无法捕获当前约 1 GiB fake record，因为垃圾长度可能仍落在 `WAL_RECORD_MAX_SIZE` 范围内。根本保护仍然是：unsynced 时不信任 header，并修复 TDE 绕过。

## 全零 header 与并发清理

发送端曾出现大量 `wal_total_len=0`，随后在 header 校验位置 core，日志中的待校验 header 字段全部为 0。可能来源包括：

1. parser 从空白/padding/已清理 buffer 中读取；
2. clean/reset 与 `check_xlog_buf()` 并发，校验线程正在读取时另一线程 `memset` CRC context；
3. 私有 DCF truncate/header-only 重建路径把无效 entry 或未填满 buffer 继续交给回调；
4. core 观察到的是 clean 后的 buffer，而不是第一次异常读取时的历史内容。

如果 clean 只由当前 append 线程在 `check_xlog_buf()` 内执行，不需要额外锁。只有 truncate/reset 等其他线程能够修改共享 node slot 或当前校验 buffer 时，才存在并发问题。

仅给 `memset` 一侧加锁没有意义。要么：

- 同一个 node 的 `load context → check_xlog_buf → save context` 全过程与 truncate clean 使用同一把锁；
- 更推荐由 truncate 线程只设置 `reset_pending` 或递增 `generation`，让对应 append 线程在下一次回调开始时清理自己的线程局部状态。

generation 方案示意：

```text
truncate线程：
  slot[node].generation++
  slot[node].reset_pending = true

append线程：
  读取generation
  发现reset_pending后自行clean
  执行check_xlog_buf
  保存状态前再次比较generation
  generation已变化则丢弃本次结果，不覆盖新状态
```

这可以避免 truncate 在线程 A 中直接清线程 B 正在使用的 TLS，也避免 append 线程在 truncate 后把旧状态重新保存回已经清理的 node slot。

定位全零问题时首先确认“header”类型：

```text
WalRecord header 全零
  → checker context/buffer clean竞争更可疑

XLogPageHeader 全零
  → 输入LSN、page定位、短读或零填充更可疑

DCF entry head(term/index/key/size/checksum)全零
  → storage truncate、entry生命周期或私有重建路径更可疑
```

## DCF truncate 后 entry 是否还会发送

### 开源实现

开源 DCF suffix truncate 不是把旧 index 填 0，而是：

```text
stream->last_index 回退
entry cache 对应槽标记 invalid
磁盘 segment 在目标 entry offset 执行 truncate
segment->last_index 回退
index_buf 只缩小逻辑 size
```

因此稳定状态下：

```text
leader自身truncate旧suffix
  → 旧index超出新的last_index
  → rep_appendlog_node不再获取或发送旧entry
  → 同一index以后重新append时，发送的是新entry

follower因term/index冲突truncate
  → leader仍保留对应entry
  → leader调整NEXT_INDEX并重新发送
```

已经在 truncate 前被 append 线程 `stg_get_entry()` 取出并增加引用的 entry 可能仍在途进入一次 CRC 回调或发送。开源 cache 通过 `valid + ref_count` 延迟释放，磁盘读取返回独立 entry，因此 truncate 不应把已借出的 `ENTRY_BUF` 直接变成全零。

如果非公开版本确实通过填 0 清理 index/header，需要额外确认：

```text
填0前是否等待entry引用归零
truncate与stg_get_entry是否使用同一生命周期保护
append线程拿到entry后，truncate是否仍能修改ENTRY_BUF(entry)
rep_appendlog_node是否在发送前重新确认index仍在有效范围
```

只要 truncate 能修改已经交给 checker 的 buffer，就必须通过 refcount、不可变快照或整段锁消除并发修改。

## 最小诊断日志

### 接收端首次调用/重启

```text
thread_id
startPtr / nbytes / startPtr % XLOG_BLCKSZ
check_end_ptr
is_first_check / is_remaining_xlog
输入前40～64字节
DCF replay index / entry key / entry len
```

### 发送端 entry

```text
thread_id / node_id / stream_id
index / entry_type / source(cache|disk)
entry_start_lsn / entry_end_lsn / entry_len
previous check_end_ptr
是否发生clean以及原因
context slot generation
```

### `check_crc()` 异常分支

```text
candidate_record_start
candidate_end_ptr
old latest_end_ptr
xl_tot_len / xl_is_new / xl_recflags
IS_RECORD_ENCRYPTED
stored_crc / calculated_crc / crc_match
is_first_check / is_remaining_xlog
ready_data_len / header_saved_len
采取动作：commit / resync / skip / panic
```

特别记录：

```text
is_first_check && IS_RECORD_ENCRYPTED && CRC_MISMATCH
```

只要捕获到该分支仍推进 `latest_end_ptr`，接收端根因即可完全闭环。

偶现问题不适合打印每条 record。建议在线程或 node slot 中保留最近 8～16 个事件的循环历史，失败时一次性输出或从 core 读取。

## Core 取证

core 只能保存崩溃时仍存在的状态，不能恢复已被覆盖的 fake record A。建议至少检查：

```gdb
thread apply all bt full
thread <walreceiverwriter-or-append-thread>
frame <check_crc-frame>

p/x $t_thrd->thrdlocal_xlog_check_cxt.latest_check_ptr
p/x $t_thrd->thrdlocal_xlog_check_cxt.latest_end_ptr
p/x $t_thrd->thrdlocal_xlog_check_cxt.check_end_ptr
p $t_thrd->thrdlocal_xlog_check_cxt.latest_wal_record_valid_size
p $t_thrd->thrdlocal_xlog_check_cxt.ready_data_len
p $t_thrd->thrdlocal_xlog_check_cxt.is_remaining_xlog
p $t_thrd->thrdlocal_xlog_check_cxt.is_first_check
x/64bx $t_thrd->thrdlocal_xlog_check_cxt.latest_wal_record
```

要回溯第一次污染位置，必须增加前述循环历史或在 TDE mismatch 静默分支保存一份固定 header 快照。

## 物理字节与 record CRC 的区分

沿相同 `[start_lsn, end_lsn)` 可以建立四个物理 hash 观测点：

```text
A. openGauss调用dcf_write前的原始WAL
B. DCF leader磁盘读取/header-only重建后的send_buf
C. follower ReceiveLogCbFunc收到的entry payload
D. walreceiverwriter实际写盘的字节
```

- 物理 hash/checksum 覆盖完整 entry 字节，包含 WAL page header 和 padding，不要求 record 对齐；
- WAL record CRC 覆盖逻辑 record，跳过物理 page header 和 padding，并按原生顺序计算 data 与 fixed header。

如果 A/B/C/D hash 一致而原生 `XLogReader` 通过，应优先定位自定义 checker；只有物理 hash 已经分叉时，才优先调查 header-only 重建、encode/decode 或 buffer 生命周期。

## 判定矩阵

| 证据 | 更可能的结论 |
| --- | --- |
| `record_start` 落在原生 record body 中 | checker 已失去 record boundary，当前 CRC 无意义 |
| `ready_data_len == fake xl_tot_len` | checker 正沿 fake record 长度稳定累计，不像单次 CRC 算法错误 |
| 重启后相同 entry/LSN/CRC 持续 core | 相同输入触发确定性状态机问题，不代表 TLS 残留 |
| 两个备机在相同位置 core | 公共输入或公共 checker bug，随机节点私有状态概率下降 |
| 原生 XLogReader/redo 通过同一 record | 自定义 checker 误报概率极高 |
| first-check encrypted mismatch 后推进 end ptr | TDE 绕过首次 resync，根因闭环 |
| `wal_total_len=0` 后发生大长度减法 | 无符号下溢导致越界读取/core |
| header 在校验中途变成全零 | 并发 clean、buffer 生命周期或私有 truncate 零填充 |
| A/B/C/D 物理 hash 一致 | DCF 传输字节可信，检查 record/page 状态机 |
| DCF entry checksum mismatch 但原生 WAL 正常 | checksum 的 buf/len/表示语义不一致 |

## 建议修复顺序

1. 修复 `is_first_check` 与 TDE 判断优先级，首次 mismatch 必须先 resync；
2. CRC 成功前不提交 `latest_end_ptr`，失败分支清理完整 fake record 状态；
3. 在所有 `wal_total_len - header_size` 前增加上下界检查；
4. 发送端不连续磁盘 entry 采用 clean + best-effort resync，unsynced 时不报错；
5. truncate/reset 改为同线程处理的 `reset_pending/generation`，或用同一把锁覆盖完整校验事务；
6. 对私有 header-only 版本确认 assembled `buf/len/key` 与最终发送完全一致；
7. 用最小日志捕获第一次 fake encrypted record，而不是只分析最终 1 GiB fake record；
8. 后续再优化首次 fake `xl_tot_len` 过大导致的大量累计和资源消耗。

## 源码坐标

### openGauss

- `src/gausskernel/storage/access/transam/xlog.cpp`
  - `CopyXLogRecordToWAL()`：record、page header 和 continuation 的物理布局；
  - `XLogWritePaxos()`：DCF entry 的原始 `buf/nBytes/end LSN`。
- `src/gausskernel/storage/access/transam/xloginsert.cpp`
  - `XLogRecordAssemble()`：形成 `xl_tot_len` 和初始 CRC。
- `src/gausskernel/storage/access/transam/xlogreader.cpp`
  - `ValidXLogRecordHeader()`；
  - `ValidXLogRecord()`。
- `src/gausskernel/storage/replication/dcf/dcf_callbackfuncs.cpp`
  - `ReceiveLogCbFunc()`；
  - DCF entry 的 `key/len/startPtr` 与 walreceiver 接入。
- `src/gausskernel/storage/replication/walreceiver.cpp`
  - `XLogWalRcvReceive()`：写入 walreceiver 环形缓冲。
- `src/gausskernel/storage/replication/walrcvwriter.cpp`
  - `walRcvWrite()`：取 `buf/nbytes/startPtr` 并写盘。

### DCF

- `src/replication/rep_leader.c`
  - `rep_appendlog_node()`：逐 index 获取 entry、拼包和发送；
  - `NEXT_INDEX` 回退、rematch 和重传；
  - `g_append_thread_id[node_id]`：node 到 append 线程的固定映射。
- `src/replication/rep_msg_pack.c`
  - `rep_encode_one_log()`：最终发送 `ENTRY_BUF/ENTRY_SIZE/key`。
- `src/storage/stream.c`
  - `stream_get_entry()`：cache 优先、disk fallback；
  - `stream_trunc_suffix()`：回退 `last_index` 并清理 suffix cache。
- `src/storage/log_storage.c`
  - `storage_get_entry()`；
  - `storage_trunc_suffix()`。
- `src/storage/segment.c`
  - `segment_get_entry()`：磁盘 entry header/data checksum；
  - `segment_trunc_suffix()`：文件 truncate 和 index buffer resize。
- `src/storage/stg_manager.h`
  - `valid/ref_count`：truncate 后已借出 entry 的生命周期保护。

## 尚待确认

- 接收端重启后第一条重发 entry 的 `index/key/len/start_lsn`，以及该 `start_lsn` 是否位于原生 record body；
- 第一次 fake record A 的 header 字节、`IS_RECORD_ENCRYPTED` 结果、CRC 和 candidate end；
- 私有版本 truncate 是否确实通过填 0 修改 index/header，以及是否可能修改已借出的 `ENTRY_BUF`；
- 发送端曾经 core 的“全零 header”究竟是 `WalRecord`、`XLogPageHeader` 还是 DCF entry head；
- first-check 跳页分支是否完整清理提前更新的 `latest_end_ptr` 和 fake record 状态；
- 私有 header-only entry 的 `original_wal_len`、stored reference len 和 assembled send len 是否始终一致。

## 相关笔记

- [DCF 模式下 XLog Entry 的切分、传输与落盘流程](dcf-xlog-transport-flow.md)
- [WAL 回放专题入口](../../postgres/replay/wal-replay-study.md)
- [主备 WAL 全链路流程图](../../postgres/replay/overview-flow.md)
- [Replay 源码地图](../../postgres/replay/source-map.md)
