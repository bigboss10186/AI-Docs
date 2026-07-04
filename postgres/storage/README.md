# PostgreSQL 存储层概念分层

这篇文档用于沉淀数据库存储层的通用概念。这里优先放 PostgreSQL 和 openGauss 都能复用的基础理解；openGauss/GaussDB 特有的 ASTORE、USTORE、CStore、D-Store 细节继续放到 `docs/gaussdb/` 下。

## 1. 先分清三条线

学习存储层时，最容易把表存储、索引存储和 WAL/XLOG 混成一个东西。它们确实会互相配合，但职责不同。

```text
表存储层:
  heap / uheap / cstore
  回答“表里的数据行或列怎么放，UPDATE/DELETE 怎么产生版本”

索引访问方法:
  btree / ubtree / gin / gist / hash ...
  回答“索引项怎么组织，怎么定位到表里的 tuple”

WAL/XLOG:
  heap WAL / btree WAL / undo WAL / xact WAL ...
  回答“上述修改怎么写日志，崩溃恢复时怎么重放”
```

所以不要把 `heap` 理解成一种 XLOG 格式。更准确地说：

```text
heap 是表数据组织和访问方法。
heap 的 INSERT/UPDATE/DELETE 会产生 heap 相关 WAL record。

btree 是索引组织和访问方法。
btree 的 insert/split/delete 会产生 btree 相关 WAL record。

WAL/XLOG 是恢复日志系统。
它记录 heap、btree、事务提交、smgr 等子系统发生过的物理/逻辑物理修改。
```

## 2. SQL 操作类型：DML、DDL、DQL、DCL、TCL

`DML` 是 Data Manipulation Language，数据操作语言。它描述的是“对表里的数据内容做增删改”的 SQL 操作。

常见 DML：

- `INSERT`：插入新行。
- `UPDATE`：修改已有行。
- `DELETE`：删除已有行。
- `MERGE`：按条件插入、更新或删除，部分数据库支持。

从存储层看，DML 是最常见的入口，因为它会直接触发表页、索引页和 WAL 的变化：

```text
INSERT
  -> 表里新增 tuple
  -> 索引里新增 index tuple
  -> 写 heap/btree WAL

UPDATE
  -> 表里产生新版本或写 undo
  -> 必要时更新索引
  -> 写 heap/uheap/undo/btree WAL

DELETE
  -> 表里标记 tuple 删除或写 undo
  -> 索引项通常不会立刻物理删除
  -> 写 heap/uheap/undo WAL
  -> 后续 VACUUM/prune/recycle 清理空间
```

和 DML 类似的 SQL 分类还有：

| 分类 | 全称 | 典型语句 | 主要含义 | 存储层关注点 |
| --- | --- | --- | --- | --- |
| DML | Data Manipulation Language | `INSERT`、`UPDATE`、`DELETE`、`MERGE` | 改表里的数据 | tuple 版本、索引维护、WAL、undo、VACUUM |
| DQL | Data Query Language | `SELECT` | 查询数据 | scan、index scan、snapshot、visibility |
| DDL | Data Definition Language | `CREATE`、`ALTER`、`DROP`、`TRUNCATE` | 改对象定义 | catalog、relfilenode、smgr、锁、WAL |
| DCL | Data Control Language | `GRANT`、`REVOKE` | 改权限 | catalog 权限元数据 |
| TCL | Transaction Control Language | `BEGIN`、`COMMIT`、`ROLLBACK`、`SAVEPOINT` | 控制事务 | CLOG/CSN、xact WAL、undo 回滚、锁释放 |

这些词是 SQL 层对操作类型的分类，不是存储引擎名字。比如：

```text
DML 是 SQL 操作类型。
heap/uheap/cstore 是表存储组织。
btree/ubtree 是索引访问方法。
WAL/XLOG 是日志与恢复系统。
```

还有一个业务开发里常见的词是 `CRUD`：

- `Create`：通常对应 `INSERT`。
- `Read`：通常对应 `SELECT`。
- `Update`：对应 `UPDATE`。
- `Delete`：对应 `DELETE`。

`CRUD` 更偏应用层说法，`DML/DQL/DDL/DCL/TCL` 更偏数据库和 SQL 分类说法。学习内核时，`DML` 这个词尤其常见，因为 INSERT/UPDATE/DELETE 会一路打到表、索引、事务、WAL、VACUUM/undo。

## 3. heap 和 uheap

`heap` 是 PostgreSQL 传统行存表的核心组织方式。它把 tuple 放在 heap page 上，用 tuple header 里的 `xmin`、`xmax`、`ctid`、infomask 等字段表达 MVCC 状态。

典型 UPDATE 可以理解为：

```text
heap_update()
  -> 旧 tuple 标记 xmax
  -> 新 tuple 插入 heap page
  -> 旧 tuple 的 ctid 指向新版本
  -> 必要时维护索引
  -> 写 heap update WAL record
```

`uheap` 是 openGauss USTORE 的 undo-based 行存组织。它不是 PostgreSQL 原生 heap 的另一个 WAL 名字，而是另一套表存储路径。它把历史版本和回滚信息更多放到 undo 中，当前 tuple、TD、undo pointer 一起参与可见性判断。

典型 UPDATE 可以理解为：

```text
UHeapUpdate()
  -> 修改 uheap tuple / page TD
  -> 写 undo record 保存旧版本或回滚信息
  -> 必要时维护 ubtree 索引
  -> 写 uheap / undo 相关 WAL record
```

一句话：

```text
heap/uheap 是“表怎么存、怎么改、怎么看”。
WAL/XLOG 是“这些修改怎么恢复”。
```

## 4. btree 和 ubtree

`btree` 是 B+Tree 索引访问方法。它不是表存储引擎，而是索引结构。对于 heap 表，btree 索引项通常保存 key，并通过 TID/ctid 指向 heap tuple。

典型索引插入可以理解为：

```text
btree insert
  -> 找到目标 leaf page
  -> 插入 index tuple
  -> 如果页面满了，执行 page split
  -> 写 btree insert/split WAL record
```

`ubtree` 是 openGauss 为 USTORE/uheap 适配的 BTree 索引路径。它仍然是索引访问方法，不是 USTORE 本身。它之所以单独存在，是因为 uheap 的 MVCC、undo、TD、可见性检查、唯一性检查和回表逻辑与传统 heap 有差异。

可以这样记：

```text
ASTORE/heap 表常见搭配 btree 索引。
USTORE/uheap 表需要适配 uheap 语义的 ubtree 索引。

heap/uheap 是表。
btree/ubtree 是索引。
heap WAL / btree WAL / undo WAL 是日志。
```

## 5. WAL record 与 rmgr

WAL record 不是统一格式的 SQL 日志。不同模块会写不同格式的 WAL payload：

- heap insert/update/delete 有 heap 自己的 WAL 格式。
- btree insert/split/delete 有 btree 自己的 WAL 格式。
- transaction commit/abort 有 xact 自己的 WAL 格式。
- relation create/truncate 有 smgr/storage 相关 WAL 格式。

PostgreSQL 用 `RmgrId` 区分这条 WAL record 属于哪个资源管理器。恢复时大致是：

```text
ReadRecord()
  -> 读到一条 WAL record
  -> 根据 RmgrId 分发
  -> heap_redo / btree_redo / xlog_redo / smgr_redo / ...
  -> redo 函数解释自己的 WAL payload
  -> 修改对应 page 或元数据
```

这也是为什么“heap”和“heap WAL”要分开理解：

```text
正常运行路径:
  heap_update() 修改 heap page，并调用 XLogInsert() 生成 WAL

崩溃恢复路径:
  recovery 读到 heap WAL record，再调用 heap_redo() 修复 heap page
```

## 6. 学习路线

建议按这个顺序理解存储层：

1. Page：数据库最基本的读写单位。
2. Tuple：行在 page 里怎么存。
3. Table access method：heap/uheap/cstore 这类表访问路径。
4. Index access method：btree/ubtree 这类索引访问路径。
5. MVCC：tuple 版本和 snapshot 可见性。
6. WAL/XLOG：修改如何持久化和恢复。
7. VACUUM/undo recycle：历史版本和空间如何清理，详见 [Page Prune、VACUUM、TRUNCATE 与空间回收](cleanup.md)。

读源码时也可以按这个顺序追：

```text
INSERT/UPDATE/DELETE
  -> 表访问方法
  -> page/tuple 修改
  -> 索引维护
  -> XLogInsert()
  -> redo 函数
```

## 7. 源码坐标

PostgreSQL：

- heap DML：`/Users/bigboss/sql/postgres/src/backend/access/heap/heapam.c`
- heap WAL/redo：`/Users/bigboss/sql/postgres/src/backend/access/heap/heapam_xlog.c`
- heap WAL 结构：`/Users/bigboss/sql/postgres/src/include/access/heapam_xlog.h`
- btree 主体：`/Users/bigboss/sql/postgres/src/backend/access/nbtree/`
- btree WAL/redo：`/Users/bigboss/sql/postgres/src/backend/access/nbtree/nbtxlog.c`
- btree 说明：`/Users/bigboss/sql/postgres/src/backend/access/nbtree/README`
- WAL 插入：`/Users/bigboss/sql/postgres/src/backend/access/transam/xloginsert.c`
- WAL 读与恢复：`/Users/bigboss/sql/postgres/src/backend/access/transam/xlogreader.c`、`/Users/bigboss/sql/postgres/src/backend/access/transam/xlogrecovery.c`
- rmgr 列表：`/Users/bigboss/sql/postgres/src/include/access/rmgrlist.h`
- 空间回收专题：[Page Prune、VACUUM、TRUNCATE 与空间回收](cleanup.md)

openGauss 对应扩展：

- heap DML：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/heap/heapam.cpp`
- UHeap DML：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uheap.cpp`
- UHeap visibility：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/knl_uvisibility.cpp`
- undo WAL：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ustore/undo/knl_uundoxlog.cpp`
- nbtree：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/nbtree/`
- ubtree：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ubtree/`
- ubtree WAL/redo：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/ubtree/ubtxlog.cpp`
- WAL 插入：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/xloginsert.cpp`
- WAL 主流程：`/Users/bigboss/sql/openGauss-server/src/gausskernel/storage/access/transam/xlog.cpp`
- rmgr 列表：`/Users/bigboss/sql/openGauss-server/src/include/access/rmgrlist.h`
