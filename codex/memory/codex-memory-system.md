# Codex 记忆系统实现分析

## 文档信息

| 字段 | 内容 |
| --- | --- |
| 主题 | Codex |
| 类型 | 源码阅读 / 机制分析 |
| 状态 | 草稿 |
| 创建时间 | 2026 年 7 月 8 日 |
| 更新时间 | 2026 年 7 月 8 日 |

## 要回答的问题

Codex 这类 agent 项目里的“记忆系统”到底是怎么实现的：

- 当前会话历史如何存储。
- 哪些历史会进入模型上下文。
- 上下文太长时如何压缩。
- 压缩后的历史如何替换原历史。
- 长期记忆如何从 thread 中抽取、落库和汇总。

这里的“记忆”不是单一模块，而是几套机制的组合：

```text
短期会话历史
  -> ContextManager.items

上下文注入与增量更新
  -> TurnContextItem / WorldState / reference_context_item / world_state_baseline

上下文压缩
  -> compact.rs / compact_remote.rs / compact_token_budget.rs

长期记忆
  -> memories DB / stage1_outputs / memory jobs / summarize_memories
```

## 结论

- Codex 的短期记忆主要是 `ContextManager` 维护的 `Vec<ResponseItem>`，它保存模型可见的对话历史、工具调用、工具输出、压缩结果等。
- Codex 不会无脑把所有运行时状态每轮都塞给模型，而是通过 `reference_context_item` 和 `world_state_baseline` 做基线对比，尽量发送增量上下文。
- 上下文压缩有多种实现：普通模型总结、远程 `/responses/compact` 压缩、以及不做总结的 token-budget 新窗口压缩。
- 压缩的关键语义不是“生成摘要”本身，而是“生成一份 replacement history，然后用它替换当前 live history”。
- 长期记忆和当前上下文压缩是两套系统。长期记忆走独立 SQLite memories DB 和后台 job pipeline；上下文压缩主要服务于当前 thread 的上下文窗口治理。

## 整体分层

可以先把 Codex 的记忆系统拆成四层：

```text
用户输入 / 工具结果 / 模型输出
        |
        v
Session.record_conversation_items()
        |
        v
ContextManager.items
        |
        +--> for_prompt() -> 本轮模型请求上下文
        |
        +--> compact_*() -> replacement history -> replace_compacted_history()
        |
        +--> rollout/thread store -> 恢复、回放、长期 memory 抽取
```

更细一点：

```text
短期上下文:
  core/src/context_manager/history.rs

上下文注入:
  core/src/session/mod.rs
  build_initial_context_with_world_state()
  record_context_updates_and_set_reference_context_item()

压缩生命周期:
  core/src/compact.rs
  core/src/compact_remote.rs
  core/src/compact_token_budget.rs

模型/服务端接口:
  core/src/client.rs
  compact_conversation_history()
  summarize_memories()

长期记忆持久化:
  state/src/runtime/memories.rs
  state/src/model/memories.rs
  state/memory_migrations/0001_memories.sql
```

## 1. 短期会话历史：ContextManager

入口文件：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/context_manager/history.rs`

核心结构：

```rust
pub(crate) struct ContextManager {
    items: Vec<ResponseItem>,
    history_version: u64,
    token_info: Option<TokenUsageInfo>,
    reference_context_item: Option<TurnContextItem>,
    world_state_baseline: Option<WorldStateSnapshot>,
}
```

字段含义：

- `items`: 当前 thread 的模型可见历史，按时间从旧到新排列。
- `history_version`: 历史被重写时递增，比如 compaction 或 rollback。
- `token_info`: token 使用信息，既依赖服务端返回，也会估算本地新增项。
- `reference_context_item`: 上一次注入给模型的 turn context 快照，用于判断下一轮是全量注入还是增量更新。
- `world_state_baseline`: 上一次注入给模型的 world state，用于生成 patch/diff。

这个结构说明 Codex 的短期记忆不是随便拼字符串，而是一组结构化 `ResponseItem`。不同类型的事件会以不同 `ResponseItem` 进入历史，例如：

- user message
- assistant message
- function/tool call
- function/tool output
- compaction item
- context update
- agent message

### 1.1 记录历史

`ContextManager::record_items()` 是把新 items 追加进历史的入口之一：

```rust
pub(crate) fn record_items<I>(&mut self, items: I, policy: TruncationPolicy)
```

它会：

1. 跳过不适合进入 API 历史的 item。
2. 调用 `process_item()` 做截断和格式处理。
3. 把处理后的 item push 到 `items`。

Session 层常用入口是：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/session/mod.rs`
- `Session::record_conversation_items()`

可以理解为：

```text
运行时事件
  -> Session.record_conversation_items()
  -> ContextManager.record_items()
  -> ContextManager.items
```

### 1.2 发送给模型前的归一化

`ContextManager::for_prompt()` 负责生成真正发送给模型的历史：

```rust
pub(crate) fn for_prompt(mut self, input_modalities: &[InputModality]) -> Vec<ResponseItem>
```

它会先调用 `normalize_history()`：

```text
ensure_call_outputs_present()
remove_orphan_outputs()
strip_images_when_unsupported()
```

这几个动作很重要：

- 工具调用必须有对应输出。
- 工具输出必须有对应调用。
- 当前模型不支持图片时，要把图片内容剥掉。

也就是说，Codex 在发送模型请求前会维护上下文结构不变量，而不是简单把历史原样发出去。

## 2. 上下文注入：不是每轮全量塞上下文

入口文件：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/session/mod.rs`

关键函数：

- `build_initial_context_with_world_state()`
- `record_context_updates_and_set_reference_context_item()`
- `start_new_context_window()`

Codex 的上下文不只是聊天记录，还包括：

- 当前配置。
- sandbox / approval / collaboration mode。
- workspace 和环境信息。
- tools / skills / MCP 等能力。
- world state。
- thread context。

如果每轮都把这些完整注入模型，会浪费上下文窗口，也会让历史快速膨胀。Codex 的做法是维护基线：

```text
reference_context_item:
  上一次注入给模型的 turn context 快照

world_state_baseline:
  上一次注入给模型的 world state 快照
```

下一轮开始时：

```text
如果没有 reference_context_item
  -> 注入完整 initial context

如果 reference_context_item 存在
  -> 比较当前 turn_context
  -> 只注入差异或必要更新
```

这套机制对 agent 项目很有启发：**上下文管理不是只有“保留多少历史”，还包括“如何避免重复注入稳定上下文”。**

## 3. 上下文压缩：三种路径

Codex 里有三类压缩：

```text
1. local responses compaction
   core/src/compact.rs

2. remote compact endpoint
   core/src/compact_remote.rs

3. token-budget compaction
   core/src/compact_token_budget.rs
```

它们的共同点是：都被建模成一次 compaction 生命周期，会触发 pre/post compact hooks，并向 UI/协议层发出 `ContextCompaction` turn item。

### 3.1 普通模型总结压缩：compact.rs

入口：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/compact.rs`
- `run_compact_task()`
- `run_inline_auto_compact_task()`

大致流程：

```text
run_compact_task()
  -> run_compact_task_inner()
  -> run_pre_compact_hooks()
  -> run_compact_task_inner_impl()
  -> clone_history()
  -> 追加 compact prompt
  -> drain_to_completed()
  -> 从本次 assistant 输出里取 summary
  -> build_compacted_history()
  -> replace_compacted_history()
  -> recompute_token_usage()
  -> run_post_compact_hooks()
```

关键逻辑在 `run_compact_task_inner_impl()`。

它会先 clone 当前历史：

```text
let mut history = sess.clone_history().await;
```

然后把 compact prompt 作为本次输入追加到临时 history：

```text
history.record_items(...)
```

之后让模型完成一次总结。总结完成后，它不是简单把 summary 存在旁边，而是构造新的历史：

```text
let mut new_history = build_compacted_history(...);
```

最后调用：

```text
sess.replace_compacted_history(...)
```

这个调用是压缩真正生效的边界。

### 3.2 远程压缩：compact_remote.rs

入口：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/compact_remote.rs`
- `run_remote_compact_task()`
- `run_inline_remote_auto_compact_task()`

远程压缩不是走普通流式对话，而是调用专门的 compact endpoint：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/client.rs`
- `compact_conversation_history()`

`client.rs` 中的注释很关键：

```text
Compacts the current conversation history using the Compact endpoint.
This is a unary call that returns a new list of ResponseItems representing the compacted transcript.
```

也就是说远程压缩流程是：

```text
当前 prompt/history
  -> /responses/compact
  -> Vec<ResponseItem>
  -> process_compacted_history()
  -> replace_compacted_history()
```

`compact_remote.rs` 还会在调用 compact endpoint 之前做一个重要处理：

```text
trim_function_call_history_to_fit_context_window()
```

如果历史里的函数输出太大，可能还没到压缩接口就已经塞不进上下文窗口。因此它会尝试把旧的 function output 改写成截断提示：

```text
"Output exceeded the available model context and was truncated"
```

这说明 agent 压缩前也需要预处理：**压缩请求自身也受上下文窗口限制。**

远程压缩返回后，Codex 会过滤 compacted history：

```text
should_keep_compacted_history_item()
```

保留：

- assistant message
- agent message
- compaction/context compaction
- 真正的 user message
- hook prompt

丢弃：

- developer message
- 非真实 user content 的 user message
- function call / function output
- reasoning
- tool search / web search / image generation 等中间项

这里的意图是：服务端返回的 compacted transcript 不能盲目信任，尤其不能保留可能过期或重复的 developer instruction。

### 3.3 Token-budget 新窗口压缩：compact_token_budget.rs

入口：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/compact_token_budget.rs`

它的注释说得很直白：

```text
Token-budget compaction skips model/server summarization and installs a fresh context window instead.
```

这类压缩不生成摘要，而是：

```text
run_compact_task_inner()
  -> run_pre_compact_hooks()
  -> emit ContextCompaction started
  -> sess.start_new_context_window()
  -> emit ContextCompaction completed
  -> run_post_compact_hooks()
```

`start_new_context_window()` 会：

1. 推进 auto compact window。
2. 重新构造 initial context。
3. 调用 `replace_compacted_history()`，把当前 history 替换成新的 context items。
4. 重新计算 token usage。

这说明 Codex 对 compaction 的抽象不是“必须总结”，而是“切换到一个新的可用上下文窗口”。

## 4. 压缩的核心语义：replacement history

无论是 local compaction、remote compaction，还是 token-budget compaction，最终都会走向同一个语义：

```text
旧 history
  -> 生成 new_history
  -> replace_compacted_history(new_history)
```

Session 层入口：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/session/mod.rs`
- `replace_compacted_history()`

`ContextManager` 里对应的是：

```rust
pub(crate) fn replace(&mut self, items: Vec<ResponseItem>) {
    self.items = items;
    self.history_version = self.history_version.saturating_add(1);
    self.world_state_baseline = None;
}
```

这意味着：

- 历史被整体替换。
- `history_version` 递增。
- world state baseline 清空，避免后续基于过期 baseline 做 diff。

这个设计对 agent 项目很重要：压缩后不是在原历史上打补丁，而是建立一个新的历史边界。

## 5. 自动压缩窗口

相关函数：

- `Session::current_window_id()`
- `Session::advance_auto_compact_window()`
- `Session::start_new_context_window()`

Codex 会给当前 thread 的上下文窗口编号：

```text
current_window_id = "{thread_id}:{window_number}"
```

每次 compaction 后会推进窗口：

```text
advance_auto_compact_window()
```

`CompactedItem` 会记录：

- `window_number`
- `first_window_id`
- `previous_window_id`
- `window_id`
- `replacement_history`

这说明 Codex 不只是替换内存状态，也会在 rollout/thread trace 中留下压缩边界，方便恢复、回放和诊断。

## 6. 长期记忆：memories DB 和后台 pipeline

长期记忆相关代码主要在 `state` crate：

- `/Users/bigboss/code/ai/codex/codex-rs/state/src/runtime/memories.rs`
- `/Users/bigboss/code/ai/codex/codex-rs/state/src/model/memories.rs`
- `/Users/bigboss/code/ai/codex/codex-rs/state/memory_migrations/0001_memories.sql`

状态运行时会打开独立 memories DB：

- `/Users/bigboss/code/ai/codex/codex-rs/state/src/runtime.rs`
- `MEMORIES_DB`
- `MemoryStore`

`0001_memories.sql` 中有两个核心表：

```sql
CREATE TABLE stage1_outputs (
    thread_id TEXT PRIMARY KEY,
    source_updated_at INTEGER NOT NULL,
    raw_memory TEXT NOT NULL,
    rollout_summary TEXT NOT NULL,
    rollout_slug TEXT,
    generated_at INTEGER NOT NULL,
    usage_count INTEGER,
    last_usage INTEGER,
    selected_for_phase2 INTEGER NOT NULL DEFAULT 0,
    selected_for_phase2_source_updated_at INTEGER
);

CREATE TABLE jobs (
    kind TEXT NOT NULL,
    job_key TEXT NOT NULL,
    status TEXT NOT NULL,
    worker_id TEXT,
    ownership_token TEXT,
    started_at INTEGER,
    finished_at INTEGER,
    lease_until INTEGER,
    retry_at INTEGER,
    retry_remaining INTEGER NOT NULL,
    last_error TEXT,
    input_watermark INTEGER,
    last_success_watermark INTEGER,
    PRIMARY KEY (kind, job_key)
);
```

可以看出长期记忆至少分两类数据：

```text
stage1_outputs:
  某个 thread 的第一阶段记忆抽取结果。

jobs:
  后台 memory 抽取和 consolidation 的任务状态。
```

`state/src/model/memories.rs` 里的 `Stage1Output` 对应 stage-1 输出：

```rust
pub struct Stage1Output {
    pub thread_id: ThreadId,
    pub rollout_path: PathBuf,
    pub source_updated_at: DateTime<Utc>,
    pub raw_memory: String,
    pub rollout_summary: String,
    pub rollout_slug: Option<String>,
    pub cwd: PathBuf,
    pub git_branch: Option<String>,
    pub generated_at: DateTime<Utc>,
}
```

这说明长期记忆不是从当前 prompt 临时算出来的，而是从历史 thread / rollout 中离线或后台抽取出来，再落到 memories DB。

## 7. 记忆摘要接口

模型客户端里还有一个专门的 memory summarization 接口：

- `/Users/bigboss/code/ai/codex/codex-rs/core/src/client.rs`
- `summarize_memories()`

它调用：

```text
/v1/memories/trace_summarize
```

注释说明：

```text
Builds memory summaries for each provided normalized raw memory.
This is a unary call (no streaming) to /v1/memories/trace_summarize.
```

这条链路和上下文压缩不同：

```text
上下文压缩:
  当前 thread 历史太长 -> replacement history -> 继续当前会话

长期记忆:
  历史 thread/rollout -> raw memory -> summarize -> memories DB / consolidation
```

所以不要把 `compact_conversation_history()` 和 `summarize_memories()` 混为一谈。前者解决上下文窗口，后者服务长期记忆资产。

## 8. 线程级 memory mode

thread 元数据里有 `memory_mode`：

- `/Users/bigboss/code/ai/codex/codex-rs/state/src/runtime/threads.rs`
- `get_thread_memory_mode()`
- `set_thread_memory_mode()`
- `mark_thread_memory_mode_polluted()`

从 `runtime/memories.rs` 的查询条件可以看到，stage-1 记忆抽取会过滤：

```text
threads.memory_mode = 'enabled'
threads.history_mode = 'legacy'
```

这说明长期记忆并不是所有 thread 都无条件生成。Codex 会按 thread 的 memory mode、history mode、更新时间、job lease、retry/backoff 等条件来选择是否抽取。

## 9. 可以借鉴的 agent 设计模式

从 Codex 这套实现里，可以抽出几个通用设计模式。

### 9.1 短期历史用结构化 item，不用纯文本拼接

Codex 的 `ContextManager.items` 是 `Vec<ResponseItem>`。这样可以保留：

- role
- tool call id
- function output
- reasoning
- compaction item
- turn id
- internal metadata

纯文本拼接虽然简单，但后面很难做：

- 工具调用/输出配对。
- 图片剥离。
- 历史归一化。
- 压缩过滤。
- rollback。
- trace/replay。

### 9.2 稳定上下文用 baseline + diff

`reference_context_item` 和 `world_state_baseline` 说明 agent 上下文应该区分：

```text
稳定上下文:
  工具、配置、环境、工作区、系统状态

动态对话:
  用户输入、模型输出、工具结果
```

稳定上下文不应该每轮全量重复注入，而应该有基线和增量。

### 9.3 压缩结果要变成新的历史边界

压缩后的产物应该是 replacement history，而不是附加说明。

推荐模型：

```text
old history
  -> compact
  -> new history
  -> history_version += 1
  -> reset stale baselines
```

### 9.4 压缩请求自身也要控上下文

`trim_function_call_history_to_fit_context_window()` 说明压缩前也可能爆上下文。真正可用的 agent 压缩系统，需要在压缩前先处理大工具输出、图片、冗余中间项。

### 9.5 长期记忆走异步 pipeline

长期记忆不应该每次对话同步重算。Codex 的 memories DB 和 jobs 表体现了一个后台 pipeline：

```text
thread 更新
  -> claim stage1 job
  -> extract raw memory
  -> mark output
  -> phase2/global consolidation
  -> 后续召回或引用
```

这比“每轮把所有历史丢给模型总结”更工程化。

## 10. 源码入口索引

相关源码入口：

1. `/Users/bigboss/code/ai/codex/codex-rs/core/src/context_manager/history.rs`
2. `/Users/bigboss/code/ai/codex/codex-rs/core/src/session/mod.rs`
3. `/Users/bigboss/code/ai/codex/codex-rs/core/src/compact_remote.rs`
4. `/Users/bigboss/code/ai/codex/codex-rs/core/src/client.rs`
5. `/Users/bigboss/code/ai/codex/codex-rs/core/src/compact_token_budget.rs`
6. `/Users/bigboss/code/ai/codex/codex-rs/state/memory_migrations/0001_memories.sql`
7. `/Users/bigboss/code/ai/codex/codex-rs/state/src/runtime/memories.rs`
8. `/Users/bigboss/code/ai/codex/codex-rs/state/src/model/memories.rs`

其中 1-4 主要对应“当前上下文怎么进模型、怎么压缩”，6-8 主要对应“长期记忆怎么落库和后台处理”。

## 还没搞懂

- 长期 memory 的 stage-1 extraction 具体由哪个 worker 调起，以及 prompt/模型请求在哪里组装。
- memories DB 里的 phase-2/global consolidation 最终如何回注到新 thread 的上下文。
- `memory_mode = polluted` 的完整状态流转和用户可见行为。
- 远程 `/responses/compact` 服务端具体如何生成 replacement history，这部分在本地源码里只能看到客户端调用和返回处理。

## 相关笔记

- [Codex Rust Workspace 分层索引](../architecture/codex-rs-index.md)
- [AI / Codex 索引](../../indexes/ai-codex.md)
- [源码阅读索引](../../indexes/source-reading.md)
