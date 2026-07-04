# AI Docs Library

这个目录作为面向 AI/Codex 协作的文档库入口。第一层目录按项目、代码库或知识域拆分，便于后续继续加入其他文档集合。

## 文档集合

- [algorithms](algorithms/README.md): 算法与工程实践中的复杂度优化、数据结构选择和解题模式。
- [gaussdb](gaussdb/README.md): GaussDB/openGauss 相关源码分析、分布式部署和故障排查笔记。
- [postgres](postgres/README.md): PostgreSQL 内核学习文档、专题阅读路径和可复用 skill。

## 组织约定

每个一层目录建议保持相对独立：

1. `README.md`: 当前文档集合的入口、阅读顺序和核心索引。
2. `topics/` 或专题目录: 存放按主题拆分的长文档。
3. `skills/`: 存放面向 AI Agent 的可复用阅读、调试或生成流程。
4. `references/`: 可选，存放外部资料、源码坐标和术语表。

新增文档集合时，优先在 `docs/` 下创建新的一级目录，再从这里加入口链接。

## 数据库内核笔记归档规则

当前数据库学习笔记按“共通概念”和“项目特有概念”分层：

1. PostgreSQL 和 openGauss/GaussDB 都适用的基础概念，优先放到 `postgres/`。例如 page、tuple、heap、btree、WAL/XLOG、rmgr、smgr、checkpoint、redo。
2. openGauss/GaussDB 特有或明显扩展的内容，放到 `gaussdb/`。例如 ASTORE、USTORE、CStore、D-Store、uheap、ubtree、undo worker、GaussDB 部署和回放差异。
3. 如果一个主题既有共通概念又有 openGauss 特化实现，先在 `postgres/` 写通用模型，再在 `gaussdb/` 写差异和源码坐标，并互相链接。
