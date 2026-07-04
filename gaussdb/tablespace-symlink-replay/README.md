# 表空间回放软链接冲突分析

## 背景

在分布式混部场景中，可能出现两个不同分片的备机部署在同一台机器上。用户在 CN 上执行绝对路径表空间 DDL 后，DDL 被下发到多个 DN：

```sql
CREATE TABLESPACE ts1 LOCATION '/data/tbs';
```

多个 DN 或 DN 备机都会处理这条表空间 DDL，但这通常不是“同一个 WAL 被多个备机重复回放”。更准确地说，是每个 DN primary 各自执行 DDL、各自生成本 DN 的表空间创建 WAL，然后对应的 DN standby 回放自己 primary 产生的 WAL。直觉上，openGauss 会在真实数据目录中拼接节点名，似乎可以避免多个实例共用同一批 relation 文件。但实际故障中，回放阶段仍可能在创建软链接时失败：

```text
could not create symbolic link ".../pg_tblspc/<tablespace_oid>": File exists
```

本文从 openGauss 源码解释为什么这种现象确实可能发生。

## 结论

openGauss 在非 DSS、PGXC 场景下，会把真实表空间版本目录写成：

```text
<LOCATION>/PG_<version>_<catalog_version>_<PGXCNodeName>
```

因此后续 relation 文件访问会进入带节点名的子目录：

```text
<LOCATION>/PG_xxx_<dn_name>/<dbOid>/<relfilenode>
```

但是表空间软链接入口本身仍然是：

```text
<TBLSPCDIR>/<tablespace_oid>
```

也就是通常意义上的：

```text
$PGDATA/pg_tblspc/<tablespace_oid>
```

这个软链接文件名不包含 DN 名。如果两个实例的 `TBLSPCDIR` 实际落到同一个物理目录，两个实例回放相同表空间 OID 时就会竞争创建同一个软链接：

```text
pg_tblspc/12345 -> /data/tbs
```

所以，问题不是后续 relation 数据路径没有带 DN 名，而是创建 `pg_tblspc/<tablespace_oid>` 这个软链接入口，或者检查 `LOCATION` 是否已被其他 symlink 使用时发生了冲突。

从当前分析看，更可能的故障模型不是“同一条 WAL 在同一个备机上被回放两遍”，也不是“两个 redo worker 同时回放同一条 CREATE TABLESPACE 记录”。正常 redo 调度下，同一条 WAL 在一个 standby 上应只处理一次；如果这里出问题，其他非幂等 redo 操作也可能更容易暴露。因此这类可能性优先级较低。

更符合低概率混部故障特征的是：多个 DN standby 在同一台机器上正常回放各自的表空间 DDL WAL，但它们使用了相同的绝对 `LOCATION`。如果真实数据目录没有拼接 DN 名，会形成严重的数据目录冲突；如果真实数据目录已经拼接 DN 名，则风险主要转移到回放前的路径检查、`LOCATION` 复用检查，或者 `pg_tblspc/<tablespace_oid>` 软链接入口是否真正隔离。

## 关键代码证据

### 1. TBLSPCDIR 是 pg_tblspc 入口目录

openGauss 中 `TBLSPCDIR` 来自实例数据目录上下文：

```c
#define TBLSPCDIR (g_instance.datadir_cxt.tblspcDir)
```

通常这个目录对应当前实例的：

```text
$PGDATA/pg_tblspc
```

如果部署或共享存储配置导致两个实例的 `TBLSPCDIR` 实际指向同一个物理目录，后续软链接入口就不再隔离。

### 2. 回放 CREATE TABLESPACE 会进入 create_tablespace_directories

表空间 WAL 类型定义：

```c
#define XLOG_TBLSPC_CREATE 0x00
#define XLOG_TBLSPC_RELATIVE_CREATE 0x20
```

redo 入口：

```c
void tblspc_redo(XLogReaderState* record)
{
    uint8 info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;

    if (info == XLOG_TBLSPC_CREATE) {
        xl_tblspc_create_rec* xlrec = (xl_tblspc_create_rec*)XLogRecGetData(record);
        xlog_create_tblspc(xlrec->ts_id, xlrec->ts_path, false);
    } else if (info == XLOG_TBLSPC_RELATIVE_CREATE) {
        xl_tblspc_create_rec* xlrec = (xl_tblspc_create_rec*)XLogRecGetData(record);
        xlog_create_tblspc(xlrec->ts_id, xlrec->ts_path, true);
    }
}
```

`xlog_create_tblspc()` 继续调用目录和软链接创建函数：

```c
void xlog_create_tblspc(Oid tsId, char* tsPath, bool isRelativePath)
{
    char* location = tsPath;

    if (isRelativePath) {
        location = reform_relative_location_under_pgdata(tsPath);
    }

    check_create_dir(location);
    create_tablespace_directories(location, tsId);
}
```

这说明 standby 回放时也会走和创建目录、创建软链接相关的代码。

### 3. 软链接路径不带 DN 名

`create_tablespace_directories()` 里首先生成 `linkloc`：

```c
static void create_tablespace_directories(const char* location, const Oid tablespaceoid)
{
    char* linkloc = palloc(strlen(TBLSPCDIR) + OIDCHARS + 2);

    sprintf_s(linkloc,
              strlen(TBLSPCDIR) + 1 + OIDCHARS + 1,
              "%s/%u",
              TBLSPCDIR,
              tablespaceoid);

    ...

    symlink(location, linkloc);
}
```

因此软链接入口是：

```text
<TBLSPCDIR>/<tablespace_oid> -> <LOCATION>
```

它不是：

```text
<TBLSPCDIR>/<tablespace_oid>_<dn_name> -> <LOCATION>
```

这就证明了软链接入口没有 DN 级命名隔离。

### 3.1 target 已被链接可能被产品代码主动判冲突

从操作系统语义看，两个不同 symlink 可以指向同一个 target：

```text
/data/dn1/pg_tblspc/12345 -> /data/tbs
/data/dn2/pg_tblspc/12345 -> /data/tbs
```

也就是说，`symlink("/data/tbs", "/data/dn2/pg_tblspc/12345")` 不会因为 `/data/tbs` 已经被别的 symlink 指向而自然返回 `EEXIST`。系统调用层面的 `EEXIST` 只表示第二个参数 `linkpath` 已经存在。

但是数据库产品可以在调用 `symlink()` 前主动扫描已有 tablespace symlink，并禁止复用同一个 `LOCATION`。openGauss 公开代码中有类似检查：

```c
static void check_tablespace_symlink(const char* location)
{
    if (t_thrd.xlog_cxt.InRecovery)
        return;

    dir = AllocateDir(TBLSPCDIR);

    while ((dent = ReadDir(dir, "pg_tblspc")) != NULL) {
        snprintf_s(tmppath, ..., "%s/%s", TBLSPCDIR, dent->d_name);

        lstat(tmppath, &st);
        if (!S_ISLNK(st.st_mode))
            ereport(ERROR, ...);

        rllen = readlink(tmppath, linkpath, sizeof(linkpath));
        linkpath[rllen] = '\0';
        canonicalize_path(linkpath);

        snprintf_s(tmppath, ..., "%s/", location);
        linkpath[rllen] = '/';
        linkpath[rllen + 1] = '\0';

        if (strncmp(tmppath, linkpath, strlen(linkpath)) == 0 ||
            strncmp(tmppath, linkpath, strlen(tmppath)) == 0) {
            ereport(ERROR,
                    errmsg("find conflict linkpath in pg_tblspc, try a different path."));
        }
    }
}
```

公开 openGauss 在 recovery 中会直接 `return`，所以它的回放路径不执行这段检查。但商业版代码可能不同：如果商业版在回放时没有跳过类似检查，或者实现了更强的跨实例 `LOCATION` 复用检查，那么即使两个 symlink 的 `linkpath` 不同，只要它们都指向同一个 `/data/tbs`，也可能被产品逻辑判定为“路径已被使用”，并返回或包装成 `EEXIST`。

### 4. 真实数据子目录带 DN 名

同一个函数中，非 DSS、PGXC 场景会构造带节点名的真实表空间版本目录：

```c
#ifdef PGXC
if (ENABLE_DSS) {
    sprintf_s(location_with_version_dir,
              ...,
              "%s/%s",
              location,
              TABLESPACE_VERSION_DIRECTORY);
} else {
    sprintf_s(location_with_version_dir,
              ...,
              "%s/%s_%s",
              location,
              TABLESPACE_VERSION_DIRECTORY,
              g_instance.attr.attr_common.PGXCNodeName);
}
#endif
```

在 recovery 中，这个带节点名的目录会被删除并重建：

```c
if (t_thrd.xlog_cxt.InRecovery) {
    if (stat(location_with_version_dir, &st) == 0 && S_ISDIR(st.st_mode)) {
        rmtree(location_with_version_dir, true);
    }
}

mkdir(location_with_version_dir, S_IRWXU);
```

也就是说，真实数据目录是按节点名隔离的。

### 5. 后续 relation 文件访问也带 DN 名

relation 路径生成时，openGauss 同样拼接 `PGXCNodeName`：

```c
snprintf_s(path,
           pathlen,
           pathlen - 1,
           "%s/%u/%s_%s/%u/%u",
           TBLSPCDIR,
           rnode.spcNode,
           TABLESPACE_VERSION_DIRECTORY,
           g_instance.attr.attr_common.PGXCNodeName,
           rnode.dbNode,
           rnode.relNode);
```

所以后续数据访问路径确实会经过带 DN 名的目录：

```text
$PGDATA/pg_tblspc/<spcOid>/PG_xxx_<dn_name>/<dbOid>/<relfilenode>
```

如果 `$PGDATA/pg_tblspc/<spcOid>` 指向 `/data/tbs`，最终就是：

```text
/data/tbs/PG_xxx_<dn_name>/<dbOid>/<relfilenode>
```

## 为什么仍然会报软链接冲突

假设两个 DN 备机在同一台机器：

```text
DN1 PGXCNodeName = dn_6001
DN2 PGXCNodeName = dn_6002
LOCATION = /data/tbs
tablespace_oid = 12345
```

如果它们的 `TBLSPCDIR` 是隔离的：

```text
/data/dn1/pg_tblspc/12345 -> /data/tbs
/data/dn2/pg_tblspc/12345 -> /data/tbs
```

后续数据目录分别是：

```text
/data/tbs/PG_xxx_dn_6001/...
/data/tbs/PG_xxx_dn_6002/...
```

这种情况下通常不会因为软链接入口冲突。

但如果两个实例的 `TBLSPCDIR` 实际共享了同一个物理目录，例如：

```text
readlink -f /data/dn1/pg_tblspc = /shared/pg_tblspc
readlink -f /data/dn2/pg_tblspc = /shared/pg_tblspc
```

那么两个实例都会创建同一个软链接文件：

```text
/shared/pg_tblspc/12345 -> /data/tbs
```

第一个实例创建成功后，第二个实例执行：

```c
symlink("/data/tbs", "/shared/pg_tblspc/12345")
```

就会因为目标路径已经存在而失败。

这与后续 relation 文件是否带 DN 名不是一回事。它失败在进入带 DN 名的数据子目录之前。

如果你已经确认两个实例的 `TBLSPCDIR` 是隔离的，例如：

```text
/data/dn1/pg_tblspc/12345 -> /data/tbs
/data/dn2/pg_tblspc/12345 -> /data/tbs
```

那么从操作系统 `symlink()` 语义看，第二个 symlink 不应该因为 target 相同而失败。若仍然返回 `EEXIST`，更可能是商业版代码做了额外检查：

```c
if (location_already_referenced_by_existing_tablespace_link(location)) {
    errno = EEXIST;
    ereport(ERROR, ...);
}
```

这种情况下，失败原因就是“产品禁止同一个绝对 `LOCATION` 被重复用作 tablespace target”，而不是“两个 DN 的真实数据目录没有带节点名隔离”。

## 需要特别注意的 recovery 行为

在 recovery 中，openGauss 会尝试移除旧的 `linkloc`：

```c
if (t_thrd.xlog_cxt.InRecovery) {
    if (lstat(linkloc, &st) < 0) {
        if (!FILE_POSSIBLY_DELETED(errno))
            ereport(ERROR, ...);
    } else if (S_ISDIR(st.st_mode)) {
        rmdir(linkloc);
    } else {
        unlink(linkloc);
    }
}

symlink(location, linkloc);
```

但如果存在并发回放、两个实例共享 `TBLSPCDIR`、文件系统状态变化，仍可能出现：

```text
检查时不存在
创建 symlink 时已经存在
```

这说明 `pg_tblspc/<tablespace_oid>` 这个路径在检查和创建之间被别的执行流创建了。

如果现场已经确认每个实例的 `pg_tblspc` 是独立的，且 `pg_tblspc/<tablespace_oid>` 本身没有残留，那么另一个重点怀疑方向就是商业版在 recovery 中也做了 `LOCATION` target 复用检查。此时即使 linkpath 不同，也会因为 target `/data/tbs` 已被其他 symlink 使用而报错。

## 现场验证方法

在出问题机器上，重点不是只看 `LOCATION`，而是看每个实例自己的 `pg_tblspc` 是否真正隔离。

### 1. 检查两个实例的 pg_tblspc 是否共享

```bash
readlink -f /path/to/dn1_pgdata/pg_tblspc
readlink -f /path/to/dn2_pgdata/pg_tblspc
```

如果输出相同，说明两个实例共享了表空间软链接入口目录，这是高风险配置。

### 2. 检查冲突的 tablespace OID 软链接

```bash
ls -l /path/to/dn1_pgdata/pg_tblspc/<tablespace_oid>
ls -l /path/to/dn2_pgdata/pg_tblspc/<tablespace_oid>
```

如果两个路径解析到同一个物理文件，说明冲突发生在软链接入口层。

### 3. 检查 LOCATION 下是否按节点名隔离

```bash
ls -l /data/tbs
```

正常情况下应看到类似：

```text
PG_xxx_dn_6001
PG_xxx_dn_6002
```

这只能证明数据子目录隔离，不代表 `pg_tblspc/<tablespace_oid>` 软链接入口也隔离。

如果想快速确认“创建绝对路径表空间后，真实数据目录有没有拼 DN 名”，可以专门创建一个测试表空间：

```sql
CREATE TABLESPACE ts_probe LOCATION '/data/tbs_probe';
```

然后在对应机器上检查：

```bash
find /data/tbs_probe -maxdepth 1 -type d -name 'PG_*' -print
```

判断标准：

```text
看到 PG_xxx_dn_6001、PG_xxx_dn_6002 这类目录：
  说明真实数据目录拼了 DN 名或其他节点唯一标识，relation 文件路径有实例隔离。

只看到 PG_xxx 这类不带节点唯一标识的目录：
  多个 DN 很可能会共用同一个 version directory，风险很高。
```

如果只创建空表空间还看不清楚，可以再创建一张小表强制落到该表空间：

```sql
CREATE TABLE ts_probe_table(id int) TABLESPACE ts_probe;
```

再检查 `/data/tbs_probe` 下的目录结构。这个方法比只看 `pg_tablespace_location()` 更可靠，因为 SQL 函数通常只返回用户指定的 `LOCATION`，不会展示内部拼接出来的 `PG_xxx_<dn_name>` 子目录。

### 4. 检查 LOCATION 本身是否是软链接

openGauss 不允许 `LOCATION` 本身是软链接：

```bash
ls -ld /data/tbs
```

如果 `/data/tbs` 是 symlink，可能触发：

```text
location "/data/tbs" is symbolic link
```

对应逻辑是：

```c
if (lstat(location, &st) == 0) {
    if (S_ISLNK(st.st_mode)) {
        ereport(ERROR,
                errmsg("location \"%s\" is symbolic link", location));
    }
}
```

### 5. 检查回放路径是否跳过 symlink 冲突检查

最快的方法是看错误文本和代码两个位置。

如果日志中出现下面这类错误：

```text
find conflict linkpath in pg_tblspc, try a different path.
```

说明执行到了 `check_tablespace_symlink()` 里的冲突检查。公开 openGauss 在 recovery 中不应该走到这里，因为函数开头会直接返回：

```c
static void check_tablespace_symlink(const char* location)
{
    ...

    if (t_thrd.xlog_cxt.InRecovery)
        return;

    dir = AllocateDir(TBLSPCDIR);
    ...
}
```

openGauss 当前对应代码位置：

```text
src/gausskernel/optimizer/commands/tablespace.cpp
```

可以用下面的方式快速定位：

```bash
rg -n "check_tablespace_symlink|InRecovery|AllocateDir\\(TBLSPCDIR\\)|find conflict linkpath" \
  src/gausskernel/optimizer/commands/tablespace.cpp
```

需要确认两件事：

```text
1. check_tablespace_symlink() 里是否仍然有 InRecovery return。
2. 扫描范围是否只是当前实例的 TBLSPCDIR，而不是跨实例扫描其他 DN 的 pg_tblspc。
```

公开 openGauss 的扫描范围是当前实例的 `TBLSPCDIR`：

```c
dir = AllocateDir(TBLSPCDIR);

while ((dent = ReadDir(dir, "pg_tblspc")) != NULL) {
    snprintf_s(tmppath, ..., "%s/%s", TBLSPCDIR, dent->d_name);
    lstat(tmppath, &st);
    readlink(tmppath, linkpath, sizeof(linkpath));
    ...
}
```

因此，如果商业版代码在 recovery 中没有 `return`，或者 `TBLSPCDIR` 被改成了共享路径/全局路径，就可能在回放时把别的实例已经创建的 symlink 当成冲突。

### 6. 检查真实数据目录是否拼接节点名的代码位置

openGauss 创建表空间真实 version directory 的逻辑也在：

```text
src/gausskernel/optimizer/commands/tablespace.cpp
```

关键代码形态是：

```c
sprintf_s(location_with_version_dir,
          ...,
          "%s/%s_%s",
          location,
          TABLESPACE_VERSION_DIRECTORY,
          g_instance.attr.attr_common.PGXCNodeName);
```

后续 relation 文件路径生成在：

```text
src/common/backend/catalog/catalog.cpp
```

关键代码形态是：

```c
snprintf_s(path,
           pathlen,
           pathlen - 1,
           "%s/%u/%s_%s/%u/%u",
           TBLSPCDIR,
           rnode.spcNode,
           TABLESPACE_VERSION_DIRECTORY,
           g_instance.attr.attr_common.PGXCNodeName,
           rnode.dbNode,
           rnode.relNode);
```

可以用下面命令快速确认商业版是否还保留类似逻辑：

```bash
rg -n "location_with_version_dir|PGXCNodeName|TABLESPACE_VERSION_DIRECTORY|RelPath|relNode" \
  src/gausskernel/optimizer/commands/tablespace.cpp \
  src/common/backend/catalog/catalog.cpp
```

## 风险判断

这类问题的关键判断如下：

```text
LOCATION 相同：
  不一定有问题，因为真实数据子目录可以靠 PGXCNodeName 隔离。

PGXCNodeName 相同：
  有严重风险，因为真实数据子目录也会冲突。

TBLSPCDIR/pg_tblspc 共享：
  有严重风险，即使 PGXCNodeName 不同，也会在 pg_tblspc/<tablespace_oid> 软链接入口冲突。

商业版检查 LOCATION 是否已被链接：
  即使 TBLSPCDIR/pg_tblspc 不共享，也可能因为两个 symlink 都指向同一个绝对 LOCATION 而被产品逻辑拒绝。

同一条 WAL 在同一 standby 上被回放两次：
  理论上也可能造成重复创建，但正常 redo 语义下概率很低；如果成立，应属于更底层的 WAL 回放调度问题。

两个 redo worker 同时回放同一条 CREATE TABLESPACE WAL：
  正常并行回放不应让同一条记录被两个 worker 同时处理，因此优先级也很低。
```

因此，“回放创建软链接时报已存在”至少有两种高优先级解释：

```text
解释 1：
  两个实例的 pg_tblspc 入口目录被共享，或者同一实例内存在并发/残留导致同一个 pg_tblspc/<tablespace_oid> 被重复创建。

解释 2：
  商业版代码主动检查 LOCATION 是否已被已有 symlink 指向，发现 /data/tbs 已被其他 tablespace link 使用，于是返回或包装成 EEXIST。
```

当前更推荐的排查优先级是：

```text
最高：
  绝对 LOCATION 在多个 DN standby 混部场景下复用，但商业版回放检查没有正确跳过或没有按实例隔离。

其次：
  真实 version directory 没有拼接 DN 名，导致多个 DN 在 /data/tbs/PG_xxx 这类目录上直接冲突。

较低：
  当前 DN 自己的 pg_tblspc/<tablespace_oid> 存在旧 symlink 残留，回放前检查先发现冲突。

极低：
  同一条 WAL 在同一 standby 上被回放两次，或两个 worker 同时回放同一条 CREATE TABLESPACE WAL。
```

## 解决方法与建议

1. 确保同机混部的每个 DN/standby 都有独立的 `PGDATA` 和独立的 `pg_tblspc` 目录。
2. 检查部署系统、DSS、共享存储、软链接、bind mount，确认没有把多个实例的 `pg_tblspc` 指到同一个物理目录。
3. 确保 `PGXCNodeName` 全局唯一，避免真实数据子目录冲突。
4. 如果商业版确实禁止 `LOCATION` target 复用，则不要让多个实例使用同一个绝对 tablespace 根路径；为每个 DN/standby 生成独立路径。
5. 从工程规避角度，最稳妥的做法是不要直接广播同一个裸绝对路径 `/data/tbs` 给所有 DN，而是在下发或执行侧为每个 DN 生成实例隔离路径，例如 `/data/tbs/dn_6001`、`/data/tbs/dn_6002`。
6. 如果允许，优先使用 openGauss 的 `RELATIVE` tablespace 模式，让路径落在实例自己的数据目录体系下。
7. 对已经失败的节点，先保留现场，再确认 `pg_tblspc/<tablespace_oid>` 是谁创建的，不要直接删除未知 symlink，避免误删仍在被其他实例使用的表空间入口。

代码排查优先看三个点：

```text
1. src/gausskernel/optimizer/commands/tablespace.cpp
   check_tablespace_symlink() 在 recovery 中是否直接 return。

2. src/gausskernel/optimizer/commands/tablespace.cpp
   create_tablespace_directories() 是否把真实目录拼成 <LOCATION>/PG_xxx_<PGXCNodeName>。

3. src/common/backend/catalog/catalog.cpp
   relation 路径生成是否继续使用 PG_xxx_<PGXCNodeName>。
```

如果商业版和公开 openGauss 不一致，尤其是 `check_tablespace_symlink()` 在 recovery 中没有跳过，或者扫描范围不再局限于当前实例的 `TBLSPCDIR`，就应该把它作为高优先级根因继续验证。

```


```

