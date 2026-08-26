# openGauss DCF 模式下 XLog CRC 偶现失败调试手册

## 文档信息

| 字段 | 内容 |
| --- | --- |
| 主题 | openGauss DCF / WAL record CRC / core 定位 |
| 类型 | 排障复盘 |
| 状态 | 可用 |
| 创建时间 | 2026 年 8 月 23 日 |
| 更新时间 | 2026 年 8 月 27 日 |

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
- 发送端第一次 core 很可能是 clean 后从 record body 中间起步，把普通 body 字节误读成小于 fixed header 的 `xl_tot_len`，随后执行 `xl_tot_len - SizeOfXLogRecord` 发生无符号下溢，最终在 `COMP_CRC32C` 内越界读取并 core。
- 这种小长度下溢具有确定性：只要重启后仍读取同一个磁盘 entry、使用同一个 `startPtr`，即使线程局部 CRC 状态已重置，也会再次读出相同的 fake `xl_tot_len`，表现为重启后持续在相同位置 core。
- 发送端第二次遇到的全零 WAL page 不一定表示 DCF 传输了损坏数据；它可能是 `XLOG_SWITCH` record 后直到当前 WAL segment 尾部的合法零填充。checker 必须识别 switch/padding 语义，不能因为 `startPtr` 对齐 page 就把零字节直接当成 page header 或 record header。

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

风险不只存在于 `wal_total_len=0`。如果 checker 从 record body 中间开始，fake header 的前四字节可能是 `1..header_size-1` 中的任意小值。`xl_tot_len` 是 `uint32`，`SizeOfXLogRecord` 又通常参与无符号运算，因此作差结果可能接近 `SIZE_MAX`；ARM 的 `COMP_CRC32C` 会把该结果作为 `Size` 直接传给 CRC 实现，最终越过有效 buffer 并 core。

原生 `XLogReader` 的顺序是：先由 `ValidXLogRecordHeader()`检查长度、rmgr 和 prev-link，再由 `ValidXLogRecord()`执行 CRC。自定义 checker 也必须保持相同的先后关系，不能把“合法 WAL 一定大于 header”作为调用 CRC 时的隐含前提。

必须在任何减法和 CRC feed 之前检查：

```cpp
if (!header_complete) {
    /* 继续收集header，不能读取长度之外的字段 */
}

uint32 header_size = is_new_record
    ? SIZE_OF_WAL_RECORD(WalRecord)
    : SIZE_OF_WAL_RECORD(OWalRecord);

if (wal_total_len < header_size ||
    wal_total_len >= WAL_RECORD_MAX_SIZE) {
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

### 为什么重启后会持续 core

如果第一次发送端 core 的输入来自磁盘 entry，则进程重启只会清除 checker 的线程局部状态，不会改变以下内容：

```text
entry index/key/len
entry中的物理WAL字节
startPtr = key - len
record body中被误读为xl_tot_len的四个字节
```

因此，只要重启后 append 线程仍从同一个 entry 开始，checker 就会再次从相同 body 偏移读取相同的小长度，并再次触发相同的无符号下溢。这种“重启后持续在同一 LSN core”更支持确定性的边界识别错误，而不是上一次进程残留的 CRC 中间态。

要完成确认，应从两次 core 或日志中对比：

```text
node_id / thread_id
index / key / entry_len / startPtr
candidate_record_start
candidate header前32～64字节
xl_tot_len / header_size
传给COMP_CRC32C的实际len
```

如果 `index/startPtr/header字节/xl_tot_len` 相同，这条根因链即可闭环。

## 全零 WAL page 与 `XLOG_SWITCH` segment 尾部填充

### 结论：零填充会同步，但它不是可解析的 WAL page

插入 `XLOG_SWITCH` record 时，openGauss 会把 `EndPos` 保留到当前 WAL segment 末尾，并确保 record 后面的空间已经在 WAL buffer 中分配和清零。`CopyXLogRecordToWAL()`按 page 更新 WAL copy status，walwriter 再把相应物理区间写盘并通过 `XLogWritePaxos()`写入 DCF。因此这些零字节属于需要保持物理布局的 WAL 数据，会正常传到 follower 并落入备机 WAL segment。

但是，这些字节只是 `XLOG_SWITCH` 后的物理 padding，不是带有合法 `XLogPageHeader` 的逻辑 WAL page：

```text
有效XLOG_SWITCH record
  → 当前page剩余部分为0
  → 后续若干完整8 KiB区域也可能全0
  → 到当前segment结束
  → 下一segment第一页重新出现合法long page header
```

原生 `XLogReader` 在成功校验 `XLOG_SWITCH` 后，不会继续把这些零页解析成 record，而是直接把 `EndRecPtr` 推到下一个 segment 起点：

```text
XLOG_SWITCH record
  → CRC成功
  → next_ptr = 当前segment结束位置
  → 跳过中间全部zero padding
  → 在下一segment读取long page header
```

因此，“主机发送了全零 page”本身不能直接证明 DCF 数据损坏。要先判断零页是否位于一个已经确认的 `XLOG_SWITCH` record 与下一个 segment 边界之间。

### 为什么只校验磁盘 entry 更容易失去 switch 上下文

发送端当前只校验 cache miss、从磁盘读取的 entry，允许中间出现未校验空洞。例如：

```text
包含XLOG_SWITCH的entry
  → cache hit
  → 未进入check_xlog_buf

后续zero-padding entry
  → cache miss，从磁盘读取
  → startPtr != check_end_ptr
  → checker clean并进入unsynced状态
  → 已不知道前面存在XLOG_SWITCH
```

此时 checker 仅凭零 page-header 前缀无法证明它是合法 switch padding，也不能把它当成可信 record。发送端的目标是 best-effort，因此应保持 unsynced、跳过当前物理 page，并在后续合法 page header/record boundary 重新同步；不能因为无法恢复上下文而报 record CRC 失败。

### 处理目标与三类状态

全零处理要同时满足：

1. 不扫描整个 8 KiB page，避免给发送线程增加明显开销；
2. 不把正常 `XLOG_SWITCH` padding 报成 record CRC 错误；
3. 不把所有零 header 都武断认定为正常 padding，避免隐藏真实读取或 WAL 问题；
4. 跳过的只是 checker 解析，原始 DCF entry 仍照常发送；
5. 跳过后不能提交 fake record 边界或残留 CRC 中间态。

建议将处理状态明确为三类。可以继续用 `is_first_check` 表示 `UNSYNCED`，但 `XLOG_SWITCH_PADDING` 还需要单独的 `skip_to_lsn`：

| 状态 | 证据 | 全零输入的处理 |
| --- | --- | --- |
| `XLOG_SWITCH_PADDING` | 已在可信 record boundary 完整收齐 record，结构和 CRC 成功，且 `rmid/info` 确认为 `XLOG_SWITCH` | 不检查中间 page，直接消费到当前 segment 末尾 |
| `UNSYNCED` / `is_first_check=true` | 初始化、entry 不连续、cache hit 空洞、reset/truncate 后尚未重新找到可信 record | 将零 header 记为“疑似 padding 或无效 page”，跳过当前 page，保持 unsynced |
| `SYNCED` / `is_first_check=false` | 前一条 record 已成功校验，下一位置由可信状态推导 | 全零 page 属于非预期物理异常；记录证据、清状态并转入 unsynced，不能继续 record CRC |

只有第一类可以确定地称为 `XLOG_SWITCH padding`。第二类虽然可以采取相同的“跳过”动作，但日志应写成 `probable padding/invalid page while unsynced`，不能宣称已经证明 padding 合法。

### 低成本判定：只检查 page header，不扫描 8 KiB

没有必要对整个 page 执行 `all-zero` 扫描。解析游标到达物理 page 边界后，先跨调用收齐 short page header；segment 第一页则收齐 long page header。随后按以下顺序判断：

```text
1. 当前是否处于已确认的XLOG_SWITCH_PADDING
   是 → 根据skip_to_lsn直接消费，不读取page内容

2. short page-header前缀是否全0
   是 → 该位置肯定不是合法XLogPageHeader
        记为zero-header/probable-padding

3. 非全0时执行正常ValidXLogPageHeader语义
   校验magic/info/pageaddr/tli/long-header字段
```

检查 `SizeOfXLogShortPHD` 范围的固定前缀只是常数级开销。即使一个 segment 尾部包含很多零页，也只读取每页几十字节，然后通过 LSN 运算跳过 page body，不需要扫描全部 8 KiB。

只检查 `xlp_magic == 0` 也能证明它不是合法 page header，但建议同时记录 short-header 前缀是否全 0，以区分：

```text
header前缀全0
  → 更像switch padding、预分配区域或truncate清零

xlp_magic错误但其他字段非0
  → 更像普通坏页、偏移错误或错误buffer
```

这里的“header 前缀全 0”仍不能证明整个 8 KiB page 都为 0。完整 page 扫描只应作为故障诊断或采样手段，不应放在发送热路径的正常分支。

### page 对齐不等于 page header 可信

下面的条件只说明解析游标位于物理 page 起点：

```cpp
startPtr % XLOG_BLCKSZ == 0
```

它不证明当前位置一定存在合法 `XLogPageHeader`，更不证明 page header 后一定是一条新 record。收齐 short/long page header 后，至少应验证：

```text
xlp_magic == XLOG_PAGE_MAGIC
xlp_pageaddr == 当前物理page LSN
xlp_info不包含XLP_ALL_FLAGS之外的位
xlp_tli符合当前校验上下文

long page header额外验证：
xlp_sysid
xlp_seg_size == XLogSegSize
xlp_xlog_blcksz == XLOG_BLCKSZ
```

校验位置应在 `check_xlog_buf()` 内部，而不是在 `rep_appendlog_node()` 调用回调之前另写一套 page parser。原因是 `check_xlog_buf()`才持有跨 entry 的 partial page-header 状态；page header 可能跨本次 `buf/nbytes`，必须先收齐再校验。推荐顺序是：

```text
check_xlog_buf(buf, nbytes, startPtr)
  → 游标到达WAL page边界
  → 跨调用收齐short/long page header
  → 校验xlp_magic/xlp_pageaddr/xlp_info等字段
  → header合法后再处理XLP_FIRST_IS_CONTRECORD/xlp_rem_len
  → 找到可信record boundary后才读取xl_tot_len和计算record CRC
```

全零 page 的 `xlp_magic` 为 0，不是合法 page header。不能先跳过一个假定的 short/long header，再从后面的零字节读取 `xl_tot_len`。处理必须发生在 page-header 层，而不是等到 record-header 或 CRC 层。

### 推荐状态机

#### 1. 输入不连续时进入 `UNSYNCED`

普通情况下：

```text
startPtr != check_end_ptr
  → 清理partial page header
  → 清理partial record/body和CRC
  → 不提交candidate/latest record end
  → is_first_check=true
  → 从本次entry重新resync
```

如果当前已经处于可信的 `XLOG_SWITCH_PADDING`，而新的 `startPtr` 仍落在同一 padding 区间 `[switch_record_end, skip_to_lsn)`，可以保留 `skip_to_lsn` 并继续跳过；如果新起点已经越过 `skip_to_lsn`，则说明下一 segment 也出现输入空洞，应回到普通 `UNSYNCED`。

#### 2. `UNSYNCED` 且 entry 从 page 中间开始

当前位置没有 page header，不能读取 `xl_tot_len`：

```text
page_off = current_lsn % XLOG_BLCKSZ
page_off != 0
  → 消费到下一page边界
  → 保持is_first_check=true
```

如果当前 entry 在下一 page 前结束，就跳过本 entry 剩余部分；下一条磁盘 entry 仍按 unsynced 处理。

#### 3. `UNSYNCED` 且 page-header 前缀全 0

```text
清理本页上的candidate/partial record状态
设置skip_to_lsn = 当前page结束LSN
消费本次buf中属于该page的部分
保持is_first_check=true
到下一page边界后重新读取page header
```

这里默认只跳当前 page，不直接跳到 segment 末尾。因为没有看到并成功校验前面的 `XLOG_SWITCH`，无法证明整个 segment 剩余空间都是 padding。逐 page 检查 header 前缀成本很低，也可以避免一个异常零页导致整个 segment 后面的合法 record 都漏检。

#### 4. page header 合法时恢复 record boundary

```text
合法page header
  |
  ├─ XLP_FIRST_IS_CONTRECORD
  |    → 根据xlp_rem_len跳过上一条record的continuation
  |    → continuation跨页时逐页收集并验证page header
  |    → 跳过MAXALIGN padding后得到candidate record boundary
  |
  └─ 无CONTRECORD
       → page header后是candidate record boundary
```

此时仍保持 `is_first_check=true`。只有第一条候选 record 的 header 合法、字节完整且 CRC 成功后，才能设置 `is_first_check=false`。

#### 5. 已确认 `XLOG_SWITCH` 时直接跳 segment

`XLOG_SWITCH` 只能在可信 record boundary、header 合法且 CRC 成功后识别，不能相信 unsynced fake header 中碰巧出现的 `rmid/info`：

```text
record CRC成功
  → rmid == RM_XLOG_ID && info == XLOG_SWITCH
  → skip_to_lsn = 向上取整到当前segment结束位置
  → 清理普通record partial状态
  → 进入XLOG_SWITCH_PADDING
```

后续输入可能在同一个 entry，也可能跨多个 entry。每次只消费：

```text
min(本次剩余nbytes, skip_to_lsn - current_lsn)
```

到达 `skip_to_lsn` 后退出 padding 状态，在下一 segment 起点收齐并验证 long page header。如果同一 buf 中已经带有下一 segment 数据，应继续处理，不能把整个 entry 一起跳掉。

#### 6. `SYNCED` 状态遇到非预期零 header

如果没有刚刚成功校验的 `XLOG_SWITCH`，可信 record 流预期位置却出现零 header，则不能把它静默标记为 padding：

```text
记录PHYSICAL_PAGE_INVALID，而不是RECORD_CRC_MISMATCH
保存page_start、期望pageaddr、header前缀、entry/index/key/len
可选：从WAL文件相同LSN重新读取一次
清理record/page/CRC中间态
is_first_check=true
best-effort跳过当前page或当前entry
DCF原entry继续发送
```

发送端当前目标是不误 core，因此可以降级并继续发送；但必须保留与 `UNSYNCED` 疑似 padding 不同的日志和计数，便于发现真正的 `original_wal_len`、truncate、旧 entry 或重建错误。

### 指针和状态更新规则

跳过零页时必须区分“物理输入已经消费”和“逻辑 record 已验证”：

| 字段 | 跳过零页时的动作 |
| --- | --- |
| `check_end_ptr` | 推进到本次实际消费的物理 LSN，用于判断下一输入是否连续 |
| `latest_end_ptr` | 不得作为成功 record end 推进 |
| `candidate_end_ptr` | 丢弃 |
| `latest_wal_record/ready_data_len` | 清零 |
| CRC 中间态 | 重新初始化 |
| partial page-header 状态 | 当前 page 跳完后清理；跨调用尚未收齐时保留 |
| `is_first_check` | 保持或重新设置为 true，直到完整 record CRC 成功 |
| `skip_to_lsn` | 零 header 时设置为 page end；已确认 switch 时设置为 segment end |

如果现有 `clean()`会把 `check_end_ptr` 也清成 INVALID，可以先保存本次消费结束 LSN，在 clean 后恢复为该物理位置；也可以让下一次调用再次走 discontinuity clean。前者日志更干净，后者实现更简单，但两种方式都不能恢复 fake record 状态。

### 参考伪代码

```cpp
while (nbytes > 0) {
    if (skip_to_lsn != InvalidXLogRecPtr && current_lsn < skip_to_lsn) {
        Size skip = Min(nbytes, skip_to_lsn - current_lsn);
        consume_physical_bytes(skip);
        continue;
    }

    if (is_first_check && current_lsn % XLOG_BLCKSZ != 0) {
        skip_to_lsn = next_page_lsn(current_lsn);
        clear_candidate_record_state();
        continue;
    }

    if (current_lsn % XLOG_BLCKSZ == 0) {
        if (!collect_complete_page_header_across_calls())
            return CHECK_NEED_MORE_DATA;

        if (page_header_prefix_is_zero()) {
            log_probable_padding_or_invalid_page();
            clear_candidate_record_state();
            is_first_check = true;
            skip_to_lsn = current_page_end_lsn();
            continue;
        }

        if (!valid_xlog_page_header(current_lsn)) {
            log_invalid_physical_page();
            clear_candidate_record_state();
            is_first_check = true;
            skip_to_lsn = current_page_end_lsn();
            continue;
        }

        consume_page_header_and_resync_contrecord();
    }

    if (!collect_complete_record_header_across_calls())
        return CHECK_NEED_MORE_DATA;

    if (xl_tot_len < header_size || xl_tot_len >= XLogRecordMaxSize) {
        log_invalid_candidate_header();
        clean_xlog_check_context();
        return CHECK_SKIPPED; /* 当前策略：直接跳过本entry的校验 */
    }

    if (!collect_record_and_check_crc())
        return CHECK_NEED_MORE_DATA;

    is_first_check = false;

    if (is_valid_xlog_switch_record()) {
        skip_to_lsn = current_segment_end_lsn();
        enter_switch_padding_state();
    }
}
```

伪代码中的 `page_header_prefix_is_zero()`只检查已收齐的 page-header 固定前缀，不扫描 page body；`is_valid_xlog_switch_record()`必须放在 CRC 成功之后。

### 分支决策汇总

```text
刚校验成功XLOG_SWITCH
  → 设置skip_to_lsn为当前segment结束位置
  → 后续输入直接消费到skip_to_lsn，不解析零页

unsynced/first-check且遇到完整全零page
  → 无法证明是合法padding，也不能当record
  → best-effort跳过该page，继续寻找可信page header

已经同步且未见XLOG_SWITCH，却在有效WAL范围遇到全零page
  → 记录PHYSICAL_PAGE_INVALID
  → 排查读取越界、truncate、旧entry或header-only重建
  → 不应把错误类型报告成record CRC mismatch
```

### 最小测试矩阵

| 用例 | 预期结果 |
| --- | --- |
| switch record、零 padding 和下一 segment 都在同一 entry | switch CRC 成功后跳到 segment end，并继续验证 long header |
| switch record 与 padding 分属多个连续磁盘 entry | `skip_to_lsn` 跨 entry 保留，不扫描零页 |
| switch entry 是 cache hit，checker 从磁盘 padding entry 开始 | zero-header 按 page 跳过，保持 unsynced，不报 CRC |
| 磁盘 entry 从零 padding page 中间开始 | 直接跳到下一 page，不把中间零字节读成 record header |
| page header 被拆到两个 entry，且二者连续 | 保存 partial page header，收齐后再判断 |
| page header partial 后发生 entry gap | 丢弃 partial header并回到 unsynced |
| synced 状态在非 segment 尾部遇到全零 header | 记录物理 page 异常、转 unsynced，不进入 CRC、不阻断发送 |
| header 前缀全零但 page body 非零 | 仍作为 invalid page 处理，日志不能断言合法 padding |
| 连续多个零页直到 segment end | 每页只检查 header 前缀；下一 segment 验证 long header |
| `XLOG_SWITCH` fake header 但 CRC 失败 | 不进入 padding 状态，按 unsynced/resync 处理 |

### 次日确认清单

重新分析第二次 core 时，优先确认：

1. 全零范围的起止 LSN，以及是否一直延续到 `XLogSegSize` 边界；
2. 下一 segment 第一页是否存在合法 long page header；
3. 零区域前最后一条有效 record 是否满足 `xl_rmid == RM_XLOG_ID && xl_info == XLOG_SWITCH`；
4. DCF entry 的 `start_lsn/end_lsn/key/len`，尤其 `end_lsn` 是否正好等于 segment 边界；
5. 包含 `XLOG_SWITCH` 的 entry 是 cache hit 还是 disk read，是否因为“只校验磁盘 entry”而漏掉；
6. checker 在全零页前是否已经发生 `startPtr != check_end_ptr` clean；
7. checker 是在 page-header 校验、record-header 校验还是 `COMP_CRC32C` 内 core；
8. 如果零页位于 segment 中部且前面没有 `XLOG_SWITCH`，再查 `original_wal_len`、磁盘读取范围、truncate 与 entry 生命周期。

## 除 `XLOG_SWITCH` 外的全零 header 与并发清理

发送端曾出现大量 `wal_total_len=0`，随后在 header 校验位置 core，日志中的待校验 header 字段全部为 0。可能来源包括：

1. `XLOG_SWITCH` 后的合法 segment 尾部 padding；
2. parser 从其他 alignment padding 或预分配文件的有效 WAL 末尾之后读取；
3. clean/reset 与 `check_xlog_buf()` 并发，校验线程正在读取时另一线程 `memset` CRC context；
4. 私有 DCF truncate/header-only 重建路径把无效 entry 或未填满 buffer 继续交给回调；
5. core 观察到的是 clean 后的 buffer，而不是第一次异常读取时的历史内容。

排查顺序应先确认 `XLOG_SWITCH + segment尾部`，再调查并发清理、truncate 和重建问题，避免把合法零填充误判成 DCF 存储损坏。

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
3. 在所有 `wal_total_len - header_size` 前增加 `SizeOfXLogRecord <= xl_tot_len < XLogRecordMaxSize` 检查，彻底阻止无符号下溢进入 CRC；
4. 在 `check_xlog_buf()` 内收齐并验证 page header；page 对齐本身不能作为 page header 或 record boundary 的证据；
5. 成功校验 `XLOG_SWITCH` 后把下一解析位置推进到 segment 边界；unsynced 状态遇到全零页时 best-effort 跳过并继续 resync；
6. 发送端不连续磁盘 entry 采用 clean + best-effort resync，unsynced 时不报错；
7. truncate/reset 改为同线程处理的 `reset_pending/generation`，或用同一把锁覆盖完整校验事务；
8. 对私有 header-only 版本确认 assembled `buf/len/key` 与最终发送完全一致；
9. 用最小日志捕获第一次 fake encrypted record，而不是只分析最终 1 GiB fake record；
10. 后续再优化首次 fake `xl_tot_len` 过大导致的大量累计和资源消耗。

## 源码坐标

### openGauss

- `src/gausskernel/storage/access/transam/xlog.cpp`
  - `CopyXLogRecordToWAL()`：record、page header 和 continuation 的物理布局；
  - `XLogWritePaxos()`：DCF entry 的原始 `buf/nBytes/end LSN`；
  - `ReserveXLogSwitch()`和 `XLogInsertRecordSingle()`：`XLOG_SWITCH` 保留并 flush 当前 segment 剩余 padding。
- `src/gausskernel/storage/access/transam/xloginsert.cpp`
  - `XLogRecordAssemble()`：形成 `xl_tot_len` 和初始 CRC。
- `src/gausskernel/storage/access/transam/xlogreader.cpp`
  - `ValidXLogRecordHeader()`；
  - `ValidXLogRecord()`；
  - `ValidXLogPageHeader()`：page magic、flags、pageaddr 与 long header 校验；
  - `XLogReadRecord()`：成功读取 `XLOG_SWITCH` 后将 `EndRecPtr` 推进到下一 segment。
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
- 发送端曾经 core 的全零 page 是否位于 `XLOG_SWITCH` 后的 segment 尾部，以及包含 switch record 的 entry 是否因 cache hit 未参与校验；
- 如果全零区域不是 switch padding，它究竟来自预分配 WAL 文件有效末尾之后、私有 truncate、旧 entry，还是 header-only body 重建长度错误；
- 第一次发送端 core 的 `xl_tot_len/header_size/COMP_CRC32C len`，以及重启前后是否命中相同 index、startPtr 和 fake header；
- first-check 跳页分支是否完整清理提前更新的 `latest_end_ptr` 和 fake record 状态；
- 私有 header-only entry 的 `original_wal_len`、stored reference len 和 assembled send len 是否始终一致。

## 相关笔记

- [DCF 模式下 XLog Entry 的切分、传输与落盘流程](dcf-xlog-transport-flow.md)
- [WAL 回放专题入口](../../postgres/replay/wal-replay-study.md)
- [主备 WAL 全链路流程图](../../postgres/replay/overview-flow.md)
- [Replay 源码地图](../../postgres/replay/source-map.md)
