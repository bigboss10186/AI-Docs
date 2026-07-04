# openGauss/GaussDB 存储引擎分析：ASTORE、USTORE、CStore、D-Store

这篇文档用于理解 openGauss/GaussDB 里的几类存储引擎形态，重点分析 ASTORE、USTORE、CStore 的概念、差异、优缺点；D-Store 由于公开资料较少，只基于 GitCode 仓库 README、配置和本地源码做谨慎分析。

参考源码：

- PostgreSQL：`/Users/bigboss/sql/postgres`
- openGauss：`/Users/bigboss/sql/openGauss-server`，分支 `dev_storage`
- D-Store：`/Users/bigboss/sql/dstore`，分支 `main`，远端 `git@gitcode.com:huaweicloud/dstore.git`

## 1. 先理解什么是存储引擎

数据库存储引擎负责把“逻辑表”变成“物理数据结构”，并处理数据读写中的核心问题：

- tuple/row 如何组织在 page 上。
- INSERT、UPDATE、DELETE 如何修改数据。
- MVCC 的历史版本放在哪里。
- snapshot 如何判断一行是否可见。
- 事务回滚如何恢复旧数据。
- WAL 如何保证崩溃恢复。
- 索引如何指向表数据。
- VACUUM/清理机制如何回收空间。

可以把存储引擎理解成执行器下面的一层：

```text
SQL / Executor
  -> Table Access Method
  -> ASTORE / USTORE / CStore / D-Store heap
  -> Buffer / Page / WAL / Undo / Tablespace
  -> 文件系统、共享存储或 page store
```

在 openGauss 里，ASTORE、USTORE 和 CStore 都属于“表的存储组织方式”，但它们解决的问题不同：

- ASTORE/USTORE 是行存，重点差异在 MVCC 历史版本放在哪里，以及 UPDATE/DELETE 如何治理膨胀。
- CStore 是列存，重点差异在数据按列组织、压缩、向量化扫描和分析查询性能。
- D-Store 的定位更大，它看起来不是简单的 `storage_type=dstore`，而是面向云原生和 page store 场景的一套独立存储内核。

更通用的存储层分层可以先看 PostgreSQL 专区的 [存储层概念分层](../../postgres/storage/README.md)。这里先给一个 openGauss 视角的速记：

```text
SQL 操作类型:
  DML -> INSERT / UPDATE / DELETE / MERGE
  DQL -> SELECT
  DDL -> CREATE / ALTER / DROP / TRUNCATE

表存储:
  ASTORE -> heap
  USTORE -> uheap
  CStore -> column store / CU

索引访问方法:
  ASTORE/heap 常见搭配 btree
  USTORE/uheap 对应适配 ubtree

日志:
  heap/uheap/btree/ubtree/undo 的修改都会写 WAL/XLOG
```

也就是说，`heap/uheap` 不是 XLOG 格式，`btree/ubtree` 也不是表存储引擎。它们的关系更像：

```text
heap/uheap/cstore 负责“表数据怎么放”。
btree/ubtree 负责“索引怎么组织并指向表数据”。
WAL/XLOG 负责“表或索引发生修改后怎么恢复”。
```

关于 `page prune`、`VACUUM`、`TRUNCATE`、`VACUUM FULL`、`freeze`、FSM/VM、autovacuum 和 USTORE undo recycle 的关系，可以继续看 PostgreSQL 专区的 [Page Prune、VACUUM、TRUNCATE 与空间回收](../../postgres/storage/cleanup.md)。

## 2. ASTORE：传统追加式行存

ASTORE 可以理解为 openGauss 中继承 PostgreSQL heap 思路的传统行存。名字里的 `A` 可以按 append-style 理解：更新时倾向于追加新版本，而不是原地覆盖旧版本。

### 2.1 核心概念

ASTORE 的核心是“多版本存在表页里”。当一行被 UPDATE 时，旧 tuple 不会马上消失，新 tuple 会被插入到 heap 中，旧 tuple 通过 `ctid` 指向新版本。DELETE 也不是立刻物理删除，而是把 tuple 标记成删除，等之后 page prune 或 VACUUM 清理。

```text
ASTORE UPDATE:

旧 tuple 仍在表页
新 tuple 追加到表页
旧 tuple 的 xmax 标记更新事务
旧 tuple 的 ctid 指向新 tuple
索引可能为新 tuple 新增索引项
VACUUM 后续回收 dead tuple
```

ASTORE/heap tuple 的 MVCC 信息主要在 tuple header：

- `xmin`：创建该 tuple 版本的事务。
- `xmax`：删除或更新该 tuple 版本的事务。
- `ctid`：当前 tuple 的位置，或指向更新后的新版本。
- infomask：记录事务状态 hint、锁、HOT 等标志。

### 2.2 UPDATE 和 HOT

ASTORE 的普通 UPDATE 是“旧版本 + 新版本”模型。它的性能问题主要来自两个地方：

- 表页上留下 dead tuple。
- 如果索引列变化，索引也要维护新条目。

PostgreSQL/openGauss heap 通过 HOT 缓解一部分问题。HOT 的条件大致是：

- 更新没有改变普通索引依赖的列。
- 新 tuple 能放在同一个 page。

满足条件时，索引仍指向 HOT chain 的 root tuple，新 tuple 作为 heap-only tuple 挂在同页链上，不需要为新版本增加普通索引项。

但要注意：HOT 只是减少索引膨胀和跨页访问，并没有改变 ASTORE 的基本模型。它仍然是追加新的 tuple 版本，不是 undo-based 原地更新。

### 2.3 可见性判断

ASTORE 的可见性判断通常直接看 tuple header 上的 `xmin/xmax`，再结合 snapshot 和事务提交状态判断：

- 创建事务是否对当前 snapshot 可见。
- 删除/更新事务是否对当前 snapshot 可见。
- 如果旧版本不可见或已被更新，是否需要沿 `ctid`/HOT chain 找新版本。

这套逻辑成熟，但会让表页承载大量历史版本。历史版本清理依赖 prune/VACUUM。

### 2.4 优点

- 成熟稳定，继承 PostgreSQL heap 的长期验证。
- 兼容性最好，生态和功能支持最完整。
- 读路径直观，tuple header 自带 MVCC 信息。
- 插入、批量导入、读多写少场景表现稳定。
- 出问题时排查资料多，工具和经验丰富。

### 2.5 缺点

- UPDATE/DELETE 会产生 dead tuple。
- 高频更新表容易膨胀。
- 索引也可能膨胀，尤其是非 HOT 更新。
- 依赖 VACUUM 回收空间，VACUUM 不及时会影响性能。
- 对强更新 OLTP 表，写放大和空间放大会比较明显。

### 2.6 适合场景

ASTORE 适合：

- 兼容性优先的业务。
- 读多写少、插入多、批量导入场景。
- 更新频率中等，且可以接受 VACUUM 管理。
- 需要使用完整 PostgreSQL/openGauss 功能面时。

## 3. USTORE：基于 undo 的行存

USTORE 是 openGauss 引入的 undo-based 行存。它的目标是缓解 ASTORE 高频更新下的表膨胀和 VACUUM 压力。

### 3.1 核心概念

USTORE 的核心是“新版本尽量留在表页，旧版本放到 undo”。它不再把所有历史版本都长期堆在 heap page 上，而是把回滚和历史可见性需要的信息写入 undo。

```text
USTORE UPDATE:

当前 tuple 尽量在表页中更新
旧版本信息写入 undo
tuple/page 上记录事务描述信息
老 snapshot 需要历史版本时，从 undo 回溯构造
事务 abort 时，也通过 undo 回滚
```

相比 ASTORE，USTORE 把历史版本压力从“表页和索引”转移到“undo 子系统”。这能降低表膨胀，但代价是可见性、回滚、undo 保留和清理都更复杂。

### 3.2 TD 和 undo

USTORE 里有 TD，即 transaction descriptor。可以粗略理解为 page 上的一组事务槽，用来记录 tuple 最近修改相关的事务信息。tuple 不是简单只靠 header 上的 `xmin/xmax` 完成所有历史判断，而是会结合 TD、undo record、事务状态进行可见性判断。

undo record 保存旧版本或回滚所需信息。一个事务产生的 undo 可以串起来；同一个 tuple/page 的历史也可以通过 undo 指针回溯。

所以 USTORE 的 MVCC 路径更像：

```text
读当前版本:
  看 tuple/page 当前状态和 TD
  判断当前事务信息对 snapshot 是否可见

需要旧版本:
  根据 undo pointer 找 undo record
  回放/构造出 snapshot 需要看到的历史 tuple
```

### 3.3 UPDATE 模型

USTORE 的 UPDATE 尽量避免 ASTORE 那种“每次都在表里新增完整历史版本”的模式。它通常会：

- 在表页保留较新的 tuple 状态。
- 把旧值、旧事务信息、回滚所需数据写入 undo。
- 在索引维护上尽量减少不必要的索引膨胀。

但 USTORE 不是“所有更新都零成本原地更新”。如果更新影响索引列，仍然要维护索引。如果 tuple 变大、页面空间不足、并发冲突复杂，也会进入更复杂路径。它解决的是 ASTORE 历史版本堆积的问题，不是消除所有写放大。

### 3.4 可见性判断

USTORE 的可见性判断比 ASTORE 更复杂。ASTORE 主要通过 tuple header 的 `xmin/xmax` 判断；USTORE 需要结合：

- tuple 当前状态。
- TD slot。
- undo pointer。
- 事务提交/回滚状态。
- 当前 snapshot。
- undo 是否仍然保留。

因此 USTORE 读“当前版本”可能很快，但读“老版本”需要访问 undo。长事务越多，undo 需要保留越久，历史版本构造成本也越需要关注。

### 3.5 VACUUM 和空间回收

ASTORE 的 VACUUM 重点清理表里的 dead tuple 和索引死项。USTORE 的表页历史版本少一些，所以表膨胀压力下降，但并不表示不需要清理。USTORE 需要处理：

- page prune。
- undo recycle。
- undo zone / undo space 元数据。
- 异步 rollback worker。
- 事务 slot 回收。

也就是说，USTORE 把“表膨胀治理”转成了“undo 生命周期治理”。长事务、复制、备机读、异常回滚都可能影响 undo 回收。

### 3.6 优点

- 更适合高频 UPDATE/DELETE 的 OLTP 场景。
- 表页不再堆积大量历史版本，表膨胀压力更小。
- VACUUM 对用户表的压力相对降低。
- 当前版本更集中，有利于更新热点表。
- 回滚信息结构化存放在 undo 中，理论上更利于事务回滚和历史版本构造。

### 3.7 缺点

- 实现复杂度明显高于 ASTORE。
- 读老版本需要访问 undo，路径更长。
- undo retention 是新的关键运维点。
- 长事务会阻碍 undo 回收。
- undo 空间、undo 元数据、undo worker 都可能成为新故障点。
- 功能兼容性不如 ASTORE 完整。

### 3.8 适合场景

USTORE 适合：

- 高频更新的 OLTP 表。
- 表膨胀和 VACUUM 压力已经明显影响业务。
- 事务较短，undo 能及时回收。
- 能接受 USTORE 当前功能限制，并提前验证兼容性。

不太适合：

- 长事务很多的系统。
- 强依赖所有 PostgreSQL/openGauss 传统功能的场景。
- 运维团队还没有监控 undo 空间和 undo 回收能力的场景。

## 4. ASTORE 与 USTORE 的核心差异

| 维度 | ASTORE | USTORE |
| --- | --- | --- |
| 基本思想 | 追加式多版本行存 | undo-based 行存 |
| 历史版本位置 | 表页中保留旧 tuple | undo 中保存旧版本/回滚信息 |
| UPDATE 行为 | 新 tuple 追加，旧 tuple 标记更新 | 尽量更新当前 tuple，旧信息入 undo |
| MVCC 入口 | tuple header 的 `xmin/xmax/ctid` | tuple/page 状态 + TD + undo |
| 读当前版本 | 直接，成熟 | 通常直接，但需结合 TD |
| 读历史版本 | 表页上找旧 tuple/HOT chain | 通过 undo 构造旧版本 |
| 表膨胀 | 高频更新下明显 | 明显缓解 |
| undo 依赖 | 弱，传统 heap 不依赖 undo 保存旧版本 | 强，MVCC 和回滚依赖 undo |
| VACUUM 压力 | 高，尤其是更新删除多时 | 表 VACUUM 压力降低，但有 undo 回收压力 |
| 索引膨胀 | 非 HOT 更新会增加索引压力 | 部分更新路径更友好，但索引列变化仍需维护 |
| 实现复杂度 | 较低 | 较高 |
| 兼容性 | 最好 | 有限制，需要验证 |
| 运维重点 | autovacuum、表/索引膨胀 | undo 空间、undo retention、回滚 worker |

可以用一句话记住：

```text
ASTORE 是“历史版本放表里，VACUUM 后面收拾”。
USTORE 是“历史版本放 undo，表里尽量保留当前状态”。
```

## 5. CStore：面向分析查询的列存

CStore 是 openGauss/GaussDB 的列式存储。它和 ASTORE/USTORE 最大的区别不是“历史版本放表里还是 undo 里”，而是“数据按行放还是按列放”。

ASTORE/USTORE 的一行通常作为一个 tuple 存在页里，适合频繁按主键访问、点查、短事务更新。CStore 则把同一列的一批值组织在一起，查询只读取需要的列，并通过压缩、min/max 粗过滤、向量化执行提升大规模扫描和聚合性能。

### 5.1 核心概念

CStore 的核心组织单位可以理解为 CU，即 compression unit。一个 CU 保存某个列的一段数据，通常会配套元数据，例如行数、NULL 信息、min/max、压缩算法信息等。

```text
行存:
  page -> row1(col1,col2,col3), row2(col1,col2,col3), ...

列存:
  column col1 -> CU1(values...), CU2(values...)
  column col2 -> CU1(values...), CU2(values...)
  column col3 -> CU1(values...), CU2(values...)
```

当查询只需要少数列时，CStore 可以少读大量无关列。比如一张宽表有 100 列，而查询只做：

```sql
select region, sum(amount)
from fact_sales
where dt >= '2026-01-01'
group by region;
```

列存主要读取 `region`、`amount`、`dt` 相关 CU；行存则更容易把整行 tuple 带上来，I/O 和解码成本更高。

### 5.2 压缩与粗过滤

CStore 对分析场景有两个天然优势：

- 同一列的数据类型一致、分布相近，更容易压缩。
- 每个 CU 可以保存 min/max 等统计信息，扫描时先做 rough check，跳过明显不满足条件的 CU。

例如 `dt` 列的某个 CU 的最大值小于查询下界，就可以整块跳过，不必解压每一行。这个思路和很多分析型数据库里的 zone map、data skipping 类似。

源码中可以看到：

- CU 与列存储主体：`cu.cpp`、`custorage.cpp`。
- 压缩：`storage/cstore/compression/`。
- min/max 和 rough check：`cstore_minmax_func.cpp`、`cstore_roughcheck_func.cpp`。

### 5.3 向量化执行

CStore 通常和向量化执行器一起使用。行存执行经常是一行一行处理 tuple；列存更适合一次处理一批列向量。

```text
传统 tuple-at-a-time:
  取一行 -> 判断 -> 聚合
  取下一行 -> 判断 -> 聚合

列存/vectorized:
  取一批 column vector
  批量判断 predicate
  批量做表达式和聚合
```

向量化执行可以减少函数调用、分支判断和 tuple deform 成本，也更容易利用 CPU cache。openGauss 的 CStore 扫描和索引扫描路径可以从 `veccstore.cpp`、`vecnodecstorescan.h`、`veccstoreindex*.cpp` 这一组代码继续读。

### 5.4 DML：INSERT、UPDATE、DELETE 的代价

CStore 不是为高频小事务更新设计的。列存对批量写入友好，但对单行 UPDATE/DELETE 往往更复杂，因为一行分散在多个列的 CU 中。

一般来说，列存更喜欢：

- 批量 INSERT / COPY。
- 分区批量加载。
- 批量删除或按分区淘汰。
- 以追加写为主，然后通过后台整理/重写优化。

openGauss CStore 中也有独立的 `cstore_insert.cpp`、`cstore_update.cpp`、`cstore_delete.cpp`、`cstore_delta.cpp`。其中 delta 相关路径可以理解为列存为了处理变更数据引入的辅助结构，避免每个小更新都直接重写大块 CU。但这不改变一个事实：高频点更新不是 CStore 的强项。

### 5.5 优点

- 分析查询性能好，尤其适合宽表、少量列扫描、聚合、过滤。
- 压缩率高，能降低存储成本和 I/O。
- CU 级 min/max、rough check 可以跳过无关数据块。
- 和向量化执行结合后，CPU 利用效率更高。
- 对批量导入、历史明细表、数据仓库事实表更友好。

### 5.6 缺点

- 点查和小范围 OLTP 查询通常不如行存直接。
- 高频 UPDATE/DELETE 成本高，变更路径比行存复杂。
- 单行写入可能带来额外缓冲、delta、合并或重写成本。
- 功能支持和限制需要单独验证，例如索引、约束、分区、ALTER、VACUUM 等。
- 排障时要同时理解 CU、压缩、delta、向量化执行器，学习成本更高。

### 5.7 适合场景

CStore 适合：

- OLAP / HTAP 中偏分析的表。
- 宽表、明细表、事实表。
- 批量导入、批量追加、少更新场景。
- 查询常常只访问少量列，并做过滤、聚合、分组。
- 历史数据、日志数据、报表数据。

不太适合：

- 高频单行 UPDATE/DELETE。
- 强事务型 OLTP 核心表。
- 频繁按主键点查并返回整行的业务。
- 需要最完整行存功能兼容性的表。

一句话记忆：

```text
CStore 是“按列组织、压缩存放、批量扫描”，目标是少读列、多跳过、批量算。
```

## 6. D-Store：资料有限下的理解

D-Store 目前公开资料比较少。GitCode 仓库 README 只明确说明它是：

```text
Dstore 是一个可独立编译，独立测试的数据库存储引擎组件。
```

从本地源码结构看，它不像 ASTORE/USTORE 那样只是表级存储格式，而是一套独立存储内核，包含：

- buffer manager
- page
- heap
- btree index
- undo
- transaction
- WAL/redo/recovery
- tablespace/segment/FSM
- lock/deadlock
- logical replication
- diagnose/tooling

### 6.1 和 ASTORE/USTORE/CStore 的关系

D-Store 的 heap 也能看到 TD、undo、inplace update、same-page append update、another-page append update 等设计，所以它在 MVCC 思路上更接近 USTORE，而不是 ASTORE。

但它和 USTORE/CStore 的区别在于：USTORE/CStore 是 openGauss 内部的表存储形态；D-Store 更像把存储层整体抽出来，形成一套可独立编译、可测试、可嵌入或适配云原生场景的存储内核。

### 6.2 云原生和透明多写线索

仓库中能看到一些和云原生方向相关的线索：

- `StorageType` 包含 `TENANT_ISOLATION`、`PAGESTORE`、`LOCAL`。
- 配置中有 `rootpdbVfsName`、`template0VfsName`、`template1VfsName`、`votingVfsName`、`runlogVfsName` 等 VFS 名称。
- `build_script/dstore_conf.json` 中有 `pagestoreEnableMultiWrite`。
- `guc.json` 中多处区分 `distribute` 和 `single` 配置。
- 工具 `pagedump`、`waldump` 支持 pagestore 场景。

这些信息说明 D-Store 大概率面向：

- 云原生 page store。
- 多租户/租户隔离。
- 存算分离或分布式存储后端。
- 透明多写，至少从配置项看有这个方向。

这里需要谨慎：目前没有看到足够公开设计文档解释“透明多写”的完整协议、数据一致性模型、故障切换策略和写入仲裁方式，所以文档中只能把它作为设计方向和源码线索，不能把细节写死。

### 6.3 初步优点

- 存储层模块完整，不只是 heap 文件格式。
- 自带 buffer/WAL/undo/transaction/tablespace/index，边界清晰。
- 有 page store 和 VFS 抽象，适合云原生存储后端。
- 有多租户、PDB、voting、runlog 等配置线索。
- 对透明多写、存算分离、分布式 page store 可能更友好。

### 6.4 初步风险

- 公开资料少，很多设计只能靠源码反推。
- 和 openGauss SQL 层的真实集成方式需要看具体集成分支。
- 透明多写涉及一致性、故障恢复、写冲突、仲裁，不能只看配置项下结论。
- 相比 ASTORE/USTORE/CStore，生态成熟度、功能兼容性、运维经验都需要验证。

## 7. 四者总览

| 维度 | ASTORE | USTORE | CStore | D-Store |
| --- | --- | --- | --- | --- |
| 定位 | 传统 heap 行存 | openGauss undo-based 行存 | 列式存储 | 独立存储引擎组件 |
| 核心目标 | 成熟稳定、兼容性 | 降低更新膨胀和 VACUUM 压力 | 分析扫描、压缩、向量化 | 云原生/独立存储内核/page store 适配 |
| 数据组织 | 一行一个 tuple | 一行一个 uheap tuple + undo | 按列分 CU | heap/page/undo/WAL 独立内核 |
| 版本管理 | 历史版本在表页 | 历史版本在 undo | 更偏批量变更/delta/重写路径 | TD + undo + 独立存储内核 |
| 更新模型 | append update | undo-based update | 小更新较重，偏批量追加 | inplace/same-page/another-page 多路径 |
| 读优势 | 通用查询、点查 | 高频更新后的当前读 | 少列扫描、聚合、过滤 | 取决于集成形态 |
| 写优势 | 插入和通用写入成熟 | 高频 UPDATE/DELETE 更友好 | 批量导入、追加写 | 面向 page store/多写场景 |
| 运维重点 | VACUUM、表/索引膨胀 | undo retention、undo 回收 | CU、压缩、delta、列存限制 | page store、透明多写、WAL/undo/checkpoint 协同 |
| 成熟度 | 最高 | 中等，需要验证限制 | 成熟但有场景边界 | 资料少，需要继续验证 |
| 适用方向 | 通用行存 | 高频更新 OLTP | OLAP/HTAP 分析表 | 云原生、存算分离、多租户、page store |

## 8. 选型建议

| 业务形态 | 推荐优先考虑 | 原因 |
| --- | --- | --- |
| 核心 OLTP 表，兼容性优先 | ASTORE | 功能面成熟，行为最接近传统 PostgreSQL heap |
| 高频更新、表膨胀明显 | USTORE | 用 undo 承担历史版本，缓解 dead tuple 堆积 |
| 宽表报表、聚合分析、批量导入 | CStore | 少读列、压缩高、向量化扫描收益明显 |
| 云原生 page store、存算分离、多写探索 | D-Store | 源码和配置显示它面向独立存储内核/page store/透明多写方向 |

最容易混淆的是：ASTORE、USTORE、CStore 是表怎么存；D-Store 更像存储层怎么被云原生化、组件化、page-store 化。前者偏“表存储格式和访问方法”，后者偏“存储内核架构”。

## 9. 源码位置索引

ASTORE / heap：

- 表访问方法入口：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/table/tableam.cpp`
- heap DML：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/heapam.cpp`
- 可见性判断：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/heapam_visibility.cpp`
- page prune / HOT：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/pruneheap.cpp`
- HOT 设计说明：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/README.HOT`
- heap 接口声明：`/Users/bigboss/sql/openGauss-server/src/include/access/heapam.h`
- nbtree 索引：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/nbtree/`
- nbtree WAL/redo：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/nbtree/nbtxlog.cpp`
- nbtree 接口声明：`/Users/bigboss/sql/openGauss-server/src/include/access/nbtree.h`

USTORE：

- USTORE table AM 回调：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/table/tableam.cpp`
- UHeap DML 主体：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uheap.cpp`
- USTORE scan：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uscan.cpp`
- USTORE page：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_upage.cpp`
- USTORE tuple：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_utuple.cpp`
- USTORE visibility：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uvisibility.cpp`
- USTORE lazy vacuum：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uvacuumlazy.cpp`
- USTORE prune：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_pruneuheap.cpp`
- undo record：`/Users/bigboss/sql/openGauss-server/src/include/access/ustore/knl_uundorecord.h`
- undo API：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/undo/knl_uundoapi.cpp`
- undo recycle：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/undo/knl_uundorecycle.cpp`
- undo WAL：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/undo/knl_uundoxlog.cpp`
- async undo worker：`/Users/bigboss/sql/openGauss-server/src/include/access/ustore/knl_undoworker.h`
- ubtree 索引：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ubtree/`
- ubtree WAL/redo：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ubtree/ubtxlog.cpp`
- ubtree 接口声明：`/Users/bigboss/sql/openGauss-server/src/include/access/ubtree.h`
- ubtree PCR：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ubtreepcr/`

ASTORE/USTORE 建表和限制：

- reloption 定义：`/Users/bigboss/sql/openGauss-server/src/include/utils/rel_gs.h`
- 建表选项处理：`/Users/bigboss/sql/openGauss-server/src/gausskernel/optimizer/commands/tablecmds.cpp`

WAL/XLOG 公共入口：

- WAL 插入：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/xloginsert.cpp`
- WAL 主流程：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/xlog.cpp`
- WAL reader：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/xlogreader.cpp`
- rmgr 分发：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/rmgr.cpp`
- rmgr 列表：`/Users/bigboss/sql/openGauss-server/src/include/access/rmgrlist.h`
- heap/nbtree/ubtree redo 相关入口：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/redo/`

CStore：

- CStore AM 主入口：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_am.cpp`
- CStore insert：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_insert.cpp`
- CStore update：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_update.cpp`
- CStore delete：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_delete.cpp`
- CStore delta：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_delta.cpp`
- CU 实现：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cu.cpp`
- CU storage：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/custorage.cpp`
- CU cache：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cucache_mgr.cpp`
- 压缩实现：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/compression/cstore_compress.cpp`
- 压缩 copy：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/compression/cstore_compress_copy.cpp`
- min/max：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_minmax_func.cpp`
- rough check：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_roughcheck_func.cpp`
- rewrite：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/cstore/cstore_rewrite.cpp`
- 向量化 CStore scan：`/Users/bigboss/sql/openGauss-server/src/gausskernel/runtime/vecexecutor/vecnode/veccstore.cpp`
- CStore index scan：`/Users/bigboss/sql/openGauss-server/src/gausskernel/runtime/vecexecutor/vecnode/veccstoreindexscan.cpp`
- CStore index heap scan：`/Users/bigboss/sql/openGauss-server/src/gausskernel/runtime/vecexecutor/vecnode/veccstoreindexheapscan.cpp`
- CStore 接口声明：`/Users/bigboss/sql/openGauss-server/src/include/access/cstore_am.h`
- CStore 结构/宏：`/Users/bigboss/sql/openGauss-server/src/include/cstore.h`
- CStore 回归测试：`/Users/bigboss/sql/openGauss-server/src/test/regress/input/hw_cstore*.source`

D-Store：

- README：`/Users/bigboss/sql/dstore/README.md`
- instance/storage config：`/Users/bigboss/sql/dstore/interface/framework/dstore_instance_interface.h`
- page store multi write 配置：`/Users/bigboss/sql/dstore/build_script/dstore_conf.json`
- heap API：`/Users/bigboss/sql/dstore/interface/heap/dstore_heap_interface.h`
- table/index smgr API：`/Users/bigboss/sql/dstore/interface/table/dstore_table_interface.h`
- heap page：`/Users/bigboss/sql/dstore/include/page/dstore_heap_page.h`
- undo type：`/Users/bigboss/sql/dstore/include/undo/dstore_undo_types.h`
- WAL type：`/Users/bigboss/sql/dstore/include/wal/dstore_wal_struct.h`
