# Page Prune、VACUUM、TRUNCATE 与空间回收

这篇文档用于理解数据库存储层里一组容易混淆的清理操作：`page prune`、`VACUUM`、`TRUNCATE`，以及相关的 `VACUUM FULL`、`freeze`、FSM、VM、autovacuum、undo recycle。

先记住一句话：

```text
page prune 是页内整理。
VACUUM 是表级垃圾回收。
TRUNCATE 是整表清空。
VACUUM FULL / CLUSTER 是重写表。
freeze 是事务号防老化。
FSM/VM 是辅助地图。
```

## 1. 为什么需要这些清理操作

MVCC 数据库不会在 `UPDATE` 或 `DELETE` 时立刻把旧版本物理删除。原因是别的事务可能还需要看到旧版本。

以 heap/ASTORE 为例：

```text
UPDATE 一行:
  旧 tuple 还留在 heap page
  新 tuple 写入 heap page
  旧 tuple 的 xmax 标记更新事务
  旧 tuple 的 ctid 指向新版本

DELETE 一行:
  tuple 不会立刻消失
  tuple header 里记录删除事务
```

当所有可能看到旧版本的事务都结束后，这些旧 tuple 才能变成真正可以回收的 dead tuple。清理系统要解决的就是：

- 哪些旧版本已经没人能看到了。
- 页内空间能不能立即复用。
- 索引里指向 dead tuple 的项什么时候删除。
- 表文件末尾的空页能不能还给操作系统。
- 事务号太老时如何避免 wraparound。

## 2. Page Prune：页内整理

`page prune` 是 heap page 内部的轻量清理。它通常发生在访问某个 page 时，发现这个 page 上有可清理的 HOT chain 或 dead tuple，于是顺手整理这个 page。

它的范围很小：

```text
作用范围:
  单个 heap page

主要目标:
  清理 HOT chain
  标记 dead line pointer
  回收 page 内部可复用空间
  缩短后续可见性判断路径
```

它不是完整 VACUUM，因为它通常不负责全表扫描，也不负责系统性清理索引死项。

### 2.1 Page Prune 处理什么

在 heap page 中，tuple 通过 line pointer 被 page item array 引用。UPDATE 多次后，可能出现类似：

```text
index -> root tuple
          |
          v
        old tuple -> newer tuple -> newest tuple
```

如果某些旧 tuple 对所有事务都不可见，prune 可以把链条整理掉：

```text
prune 前:
  root -> old1 -> old2 -> live

prune 后:
  root redirect -> live
  old1/old2 标记 dead 或 unused
  page 内空间可复用
```

### 2.2 Prune 和 VACUUM 的区别

| 维度 | Page Prune | VACUUM |
| --- | --- | --- |
| 粒度 | 单个 page | 整张表或多个 page |
| 触发 | 访问 page 时顺手做，或 VACUUM 扫描时做 | 手动 `VACUUM` 或 autovacuum |
| 主要对象 | heap page 内 HOT/dead tuple | heap dead tuple、索引死项、统计信息、freeze |
| 是否清索引 | 通常不系统性清索引 | 会配合 index AM 删除死索引项 |
| 目的 | 页内空间复用、缩短 HOT chain | 控制膨胀、维护统计、推进冻结 |

简单说：prune 是“这个 page 脏了，我先把页内链条理顺”；VACUUM 是“这张表积累垃圾了，我系统扫一遍”。

## 3. VACUUM：表级垃圾回收

普通 `VACUUM` 的目标不是把表文件完全缩小，而是让表内部空间可以复用，并清理索引里的死引用。

典型流程可以理解为：

```text
VACUUM table
  -> 计算 OldestXmin / cutoff
  -> 扫描 heap page
  -> 判断 tuple 是否 dead / recently dead / live
  -> prune page
  -> 收集 dead tuple TID
  -> 调用 index AM 删除相关索引项
  -> 回到 heap page 删除 dead line pointer
  -> 更新 FSM/VM
  -> 必要时 freeze tuple
  -> 尝试截断表尾空页
```

### 3.1 Dead tuple 和 recently dead

VACUUM 不能只看“这行被删了”就立刻清理。它必须判断是否还有事务可能看到旧版本。

常见状态可以粗略理解为：

- live：当前仍然可见或未来可能可见，不能删。
- dead：所有事务都不可能再看到，可以清理。
- recently dead：已经被删除/更新，但仍可能被某些老 snapshot 看到，暂时不能清理。

这就是长事务会导致表膨胀的原因：

```text
长事务持有很老的 snapshot
  -> OldestXmin 无法推进
  -> 很多旧 tuple 只能保持 recently dead
  -> VACUUM 不能删
  -> 表和索引继续膨胀
```

### 3.2 VACUUM 清 heap 和清 index

heap dead tuple 被清掉后，索引里可能仍有指向这些 tuple 的 index tuple。如果不清索引，索引扫描会不断命中无效 TID，再回表发现不可见，性能和空间都会变差。

所以 VACUUM 需要和索引访问方法协作：

```text
heap scan 找到 dead tuple TID
  -> btree / gin / gist / hash 等 index AM bulk delete
  -> 删除索引里指向 dead tuple 的条目
  -> heap page 最终回收 line pointer / 空间
```

这也是为什么索引有自己的 vacuum 回调，btree 也会有 btree vacuum 相关 WAL。

### 3.3 VACUUM 和表文件大小

普通 `VACUUM` 通常不会把表文件中间的空洞还给操作系统。它主要是把空间记录为可复用：

```text
表文件:
  page 1: 有数据
  page 2: 有空洞，VACUUM 后可复用
  page 3: 有数据
  page 4: 空
  page 5: 空

普通 VACUUM:
  page 2 的空间放进 FSM，后续 INSERT 可复用
  如果 page 4/5 位于表尾，可能尝试 truncate relation 尾部
```

如果空洞在文件中间，普通 VACUUM 不会移动 page，所以文件大小不一定下降。

## 4. VACUUM FULL：重写表

`VACUUM FULL` 和普通 `VACUUM` 不是一回事。

普通 `VACUUM` 是原地清理：

```text
原表文件不整体重写
空洞留在内部，供后续复用
通常不显著缩小文件
锁相对轻一些
```

`VACUUM FULL` 是重写表：

```text
创建新的紧凑表文件
把仍然有效的 tuple 拷贝过去
重建索引
切换 relfilenode
旧文件删除
需要更重的锁
可以真正缩小文件
```

可以把它理解成“把活数据倒到一个新表文件里”。它适合严重膨胀后的离线治理，但不适合作为高频日常操作。

## 5. TRUNCATE：整表清空

`TRUNCATE` 是 DDL，语义是快速清空整张表。它不是逐行 `DELETE`，也不需要逐个 tuple 判断可见性。

```text
DELETE FROM t:
  对每行产生删除标记
  写大量行级 WAL
  后续还需要 VACUUM 回收
  可触发 DELETE trigger

TRUNCATE t:
  直接清空 relation 存储
  通常切换或截断物理文件
  写 TRUNCATE 相关 WAL
  需要更强锁
  触发 TRUNCATE trigger
```

所以 TRUNCATE 快，是因为它绕过了逐行 MVCC 删除路径。

### 5.1 TRUNCATE 为什么需要强锁

TRUNCATE 会让整张表的数据瞬间消失，并且通常影响索引、TOAST、序列重启、外键依赖、逻辑复制等对象。它不能和普通读写并发得太松。

需要关注：

- 是否有外键引用。
- 是否 `CASCADE`。
- 是否 `RESTART IDENTITY`。
- 是否有触发器。
- 是否参与复制或逻辑解码。
- 是否有长事务依赖旧数据语义。

### 5.2 TRUNCATE 和 VACUUM 的关系

TRUNCATE 不需要后续 VACUUM 才能清空这张表，因为它不是逐行制造 dead tuple。

但 TRUNCATE 和 VACUUM 都可能涉及 relation 文件大小变化：

| 操作 | 是否逐行处理 | 是否产生 dead tuple | 是否释放文件空间 | 锁 |
| --- | --- | --- | --- | --- |
| `DELETE` | 是 | 是 | 否，需 VACUUM 后复用 | 行锁/表级轻锁 |
| `VACUUM` | 扫描页和 tuple | 清理已有 dead tuple | 通常只复用，表尾可能截断 | 较轻 |
| `VACUUM FULL` | 搬迁 live tuple | 清理并重写 | 是 | 重 |
| `TRUNCATE` | 否 | 否 | 是，整表清空 | 重 |

## 6. Freeze：事务号防老化

PostgreSQL 的事务 ID 是有限长度，会发生 wraparound。为了避免非常老的 tuple 的 `xmin` 在事务号回绕后被误判，VACUUM 还要做 freeze。

freeze 可以粗略理解为：

```text
某个 tuple 是非常老的已提交版本
  -> 不需要再保留具体创建 xid
  -> 把它标记成 frozen
  -> 未来所有事务都可以认为它已提交且可见
```

这和空间回收不是同一件事，但 VACUUM 经常同时处理两者：

- 清 dead tuple 是为了回收空间。
- freeze live tuple 是为了防止事务号回绕。

所以即使一张表几乎没有删除更新，也可能因为 freeze 需要被 autovacuum 扫描。

## 7. FSM 和 VM

### 7.1 FSM：Free Space Map

FSM 记录 relation 中哪些 page 有可用空间，帮助 INSERT 快速找到能放 tuple 的 page。

```text
VACUUM / prune 发现 page 有空闲空间
  -> 更新 FSM

INSERT 需要空间
  -> 查 FSM
  -> 找到可能有足够空间的 page
```

FSM 是性能辅助结构，不是 MVCC 正确性的核心。

### 7.2 VM：Visibility Map

VM 记录 page 是否 all-visible / all-frozen。

- all-visible：这个 page 上的 tuple 对所有事务都可见。
- all-frozen：这个 page 上的 tuple 已经冻结，不再有事务号老化风险。

VM 很关键，因为它影响：

- index-only scan 能否不回表。
- VACUUM 能否跳过某些 page。
- freeze 进度判断。

INSERT/UPDATE/DELETE 可能清 VM bit，因为 page 新增了未必对所有事务可见的 tuple。

## 8. Autovacuum：自动清理

`autovacuum` 是后台自动执行 VACUUM/ANALYZE 的机制。它的目标是避免用户必须手工清理每张表。

触发因素通常包括：

- dead tuple 数量超过阈值。
- 插入量较大，需要 analyze 更新统计信息。
- 表的 relfrozenxid 太老，需要防 wraparound vacuum。
- TOAST 表也需要清理。

autovacuum 不只是性能优化，它也关系到数据库安全运行。防 wraparound vacuum 如果被长期阻塞，风险会越来越高。

## 9. USTORE/undo 场景的差异

在 openGauss USTORE 里，历史版本主要进入 undo，而不是像 ASTORE/heap 那样大量堆在表页里。

所以清理重点会变成两条线：

```text
表页清理:
  uheap page prune
  uheap vacuum
  ubtree 清理

undo 清理:
  undo record 是否还被老 snapshot 需要
  undo space 是否能 recycle
  rollback worker / undo worker 是否正常推进
```

这意味着 USTORE 能缓解表膨胀，但不会消灭清理问题。它把很多压力从“表页 dead tuple + VACUUM”转移到“undo retention + undo recycle”。

长事务在 USTORE 下仍然危险：

```text
长事务保留老 snapshot
  -> 老版本 undo 不能回收
  -> undo 空间增长
  -> 老版本查询需要更长 undo 回溯
```

## 10. 常见误区

| 误区 | 更准确的理解 |
| --- | --- |
| DELETE 会立刻释放磁盘空间 | DELETE 主要产生 dead tuple，后续 VACUUM 才能回收复用 |
| VACUUM 一定会缩小表文件 | 普通 VACUUM 主要复用内部空间，只有表尾空页可能截断 |
| VACUUM FULL 就是更强的 VACUUM | VACUUM FULL 是重写表，锁和代价都更重 |
| TRUNCATE 等于 DELETE 全表 | TRUNCATE 是 DDL，绕过逐行 MVCC 删除，直接清空 relation |
| page prune 等于 VACUUM | prune 是页内局部整理，VACUUM 是表级系统清理 |
| USTORE 不需要清理 | USTORE 仍需要 uheap prune/vacuum 和 undo recycle |
| 索引会自动马上删除死项 | 很多索引死项依赖 VACUUM 或索引清理流程 |

## 11. 概念清单

这个专题涉及以下关键概念：

1. `UPDATE/DELETE` 如何产生旧版本。
2. `HeapTupleSatisfiesVacuum` 如何判断 tuple 能不能清理。
3. `page prune` 如何整理单个 page。
4. `VACUUM` 如何扫描 heap 并收集 dead TID。
5. index AM 如何删除死索引项。
6. VM/FSM 如何被维护。
7. freeze 如何推进 relfrozenxid。
8. `TRUNCATE` 如何绕过逐行删除，直接清空 relation。
9. USTORE 下 undo recycle 如何替代一部分 heap 历史版本清理压力。

## 12. 源码坐标

PostgreSQL：

- page prune 声明：`/Users/bigboss/sql/postgres/src/include/access/heapam.h`
- page prune 实现：`/Users/bigboss/sql/postgres/src/backend/access/heap/pruneheap.c`
- heap VACUUM：`/Users/bigboss/sql/postgres/src/backend/access/heap/vacuumlazy.c`
- VACUUM 命令入口：`/Users/bigboss/sql/postgres/src/backend/commands/vacuum.c`
- VACUUM 公共头文件：`/Users/bigboss/sql/postgres/src/include/commands/vacuum.h`
- TRUNCATE 命令：`/Users/bigboss/sql/postgres/src/backend/commands/tablecmds.c`
- heap truncate/catalog 接口：`/Users/bigboss/sql/postgres/src/include/catalog/heap.h`
- heap prune/truncate/freeze WAL 结构：`/Users/bigboss/sql/postgres/src/include/access/heapam_xlog.h`
- heap redo：`/Users/bigboss/sql/postgres/src/backend/access/heap/heapam_xlog.c`
- visibility map：`/Users/bigboss/sql/postgres/src/backend/access/heap/visibilitymap.c`
- visibility map 接口：`/Users/bigboss/sql/postgres/src/include/access/visibilitymap.h`
- free space map：`/Users/bigboss/sql/postgres/src/backend/storage/freespace/`
- btree vacuum/WAL：`/Users/bigboss/sql/postgres/src/backend/access/nbtree/nbtxlog.c`
- index vacuum API：`/Users/bigboss/sql/postgres/src/include/access/genam.h`

openGauss：

- heap prune：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/pruneheap.cpp`
- heap VACUUM：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/vacuumlazy.cpp`
- USTORE prune：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_pruneuheap.cpp`
- USTORE VACUUM：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uvacuumlazy.cpp`
- undo recycle：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/undo/knl_uundorecycle.cpp`
- undo worker：`/Users/bigboss/sql/openGauss-server/src/include/access/ustore/knl_undoworker.h`
- VACUUM 命令：`/Users/bigboss/sql/openGauss-server/src/gausskernel/optimizer/commands/vacuum.cpp`
- VACUUM FULL/CLUSTER 重写：`/Users/bigboss/sql/openGauss-server/src/gausskernel/optimizer/commands/cluster.cpp`
- TRUNCATE 命令：`/Users/bigboss/sql/openGauss-server/src/gausskernel/optimizer/commands/tablecmds.cpp`
- TimeCapsule truncate：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/tcap/tcap_truncate.cpp`
