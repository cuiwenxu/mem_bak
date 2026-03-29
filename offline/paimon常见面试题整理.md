# Apache Paimon 常见面试题整理

## 一、先用一句话讲清楚 Paimon

Paimon 是一个面向实时湖仓场景的表格式存储系统，核心能力是把流式写入、主键更新、快照管理、元数据管理和多引擎查询整合在一起。它既能做追加型表，也能做主键表；主键表底层是 LSM 结构，通过 snapshot + manifest + data file 组织数据和元数据。

## 二、高频面试题地图

面试里最常见的问题通常集中在 6 个方向：

1. Paimon 的定位与适用场景
2. 主键表、Append 表、Bucket、Compaction 等存储模型
3. Snapshot / Manifest / Schema 等元数据结构
4. Catalog、系统表、`sys` 数据库
5. Flink SQL 里的 `CALL`、Procedure、Action Jar
6. 运维与故障恢复：回滚、修复、孤儿文件、并发冲突

---

## 三、常见面试题与标准回答

### 1. Paimon 是什么？适合什么场景？

**标准回答**

Paimon 可以理解为“面向实时更新场景的湖仓表存储”。它特别适合 CDC 入湖、宽表汇总、主键去重、明细+状态表统一存储这类场景。相比只擅长离线分析的表格式，它更强调流式写入、主键更新、增量消费和实时可见。

**面试官可能追问**

- 为什么 CDC 场景更适合 Paimon？
- 它和 Iceberg / Hudi 的主要差异是什么？

**答题关键词**

实时湖仓、主键表、流批一体、CDC、增量消费

### 2. Paimon 的核心表类型有哪些？

**标准回答**

最重要的是两类：

- Append Table：适合纯追加数据，读取时不需要做主键合并。
- Primary Key Table：适合 upsert / delete / CDC 场景，底层会按主键维护多层文件，查询时通常要做 merge。

主键表的实现重点不是“覆盖文件”，而是“通过 LSM 结构逐步合并”。

### 3. Paimon 主键表为什么常被说成是 LSM？

**标准回答**

因为它不是每次更新都改写整张大文件，而是不断生成新的分层文件，再通过 compaction 合并。每个 bucket 里可以看成一棵局部 LSM tree。这样写入吞吐更好，但查询侧要承担一定 merge 成本，所以 compaction 对读性能影响很大。

### 4. Snapshot、Manifest、Data File 的关系是什么？

**标准回答**

这是 Paimon 元数据链路里最重要的一题。

- Snapshot：某次提交后的全局版本入口。
- Manifest List：记录这次快照引用了哪些 manifest。
- Manifest File：记录具体新增/删除了哪些 data file。
- Data File：真正存数据的文件。

也就是说，读取链路一般是 `snapshot -> manifest list -> manifest -> data file`。排查元数据、定位文件引用关系、做恢复分析时，都是沿着这条链往下追。

### 5. Bucket 有哪些模式？为什么重要？

**标准回答**

Bucket 是 Paimon 读写的最小物理组织单元。常见理解有三类：

- 固定 bucket：`bucket > 0`，按 hash 分配，容易理解，但后续扩缩桶是离线运维问题。
- 动态 bucket：主键表默认常见模式，`bucket = -1`，Paimon 维护 key 到 bucket 的映射，能自动扩容。
- Postpone / unaware bucket 等能力在新版本里也有，但面试里最常问的还是固定和动态。

Bucket 数量直接影响并行度、小文件、写入热点和后续恢复复杂度。

### 6. Compaction 为什么重要？

**标准回答**

因为主键表会不断积累 sorted runs，如果不做 compaction，读取时要 merge 的文件越来越多，查询性能会明显下降，严重时还会带来内存压力。Compaction 的本质是拿写放大换读性能。

**一句加分回答**

Paimon 的运维重点之一其实就是控制 compaction 节奏，避免“小文件太多”和“compaction 过重”两个极端。

---

## 四、元数据专题高频题

### 7. Paimon 的元数据存在哪里？

**标准回答**

表的底层目录里会有一套元数据文件和数据文件组织。核心元数据包括：

- schema 相关文件
- snapshot 目录
- manifest / manifest list
- index / statistics 等辅助文件

真正存在哪个物理存储上，取决于表路径和 catalog 配置，比如 HDFS、对象存储、本地文件系统等。

### 8. Paimon 元数据怎么修复？

**标准回答**

面试里不要直接回答“手工改文件”，标准思路应该是：

1. 先判断损坏层级：是 data file、manifest、snapshot，还是 metastore/catalog 信息不一致。
2. 优先用官方能力恢复，比如 `rollback` / `rollback_to`、`repair`、重新同步 catalog 元信息。
3. 如果是文件层损坏，要沿 `snapshot -> manifest -> data file` 找影响范围，确认哪些快照引用了坏文件。
4. 再决定是回滚到某个可用 snapshot，还是重建表 / 重灌数据 / 用备份恢复。

**面试加分点**

“元数据修复”的本质不是把一个 JSON 改对，而是恢复“快照引用链的一致性”。

### 9. 能不能只修复单个 bucket 的文件？

**标准回答**

通常不能把它当成一个完全独立的问题来处理，因为 bucket 文件是否可恢复，要看它是否还被 manifest / snapshot 引用。单个 bucket 文件损坏后，关键不是“这个文件在不在”，而是“当前快照链是否还能正确引用并读取它”。

如果只是某个历史文件坏了，但最新可见 snapshot 已经不再引用它，影响可能有限；如果当前活跃 snapshot 还引用它，就要考虑回滚、重放、补写或从备份恢复。

### 10. 为什么恢复时要先查 bucket 文件到 snapshot 的映射？

**标准回答**

因为只有知道一个坏文件被哪些 snapshot 引用，才能判断：

- 当前线上读是否受影响
- 回滚到哪个 snapshot 是安全的
- 哪些下游增量消费会被波及
- 是否存在更低成本的恢复点

这个问题本质是在问你是否理解 Paimon 的快照引用链。

### 11. orphan files 是什么？怎么检测和清理？

**标准回答**

orphan files 指文件系统里还存在，但已经不被任何 Paimon snapshot 引用的文件。出现原因通常是异常删除失败、任务中断、提交过程遗留等。

官方提供了 `remove_orphan_files` 过程来清理。生产上要注意：

- 默认会设置时间保护，避免误删刚写入但尚未稳定可见的文件。
- 最好先 dry run，再正式删除。
- 清理前先确认没有长时间运行但未提交完成的写作业。

---

## 五、系统表与 Catalog 高频题

### 12. Paimon 系统表在哪？和 catalog 类型有关吗？

**标准回答**

系统表不是“单独的一套真实物理表”，而是 Paimon 对元数据暴露出来的查询视图。常见访问方式有两类：

- 表级系统表：`my_table$snapshots`、`my_table$schemas`、`my_table$files`、`my_table$manifests` 等
- 全局系统表：通过 `sys` 数据库访问，比如 `sys.all_table_options`、`sys.catalog_options`

**和 catalog 类型的关系**

- 有关系，但不是“有没有系统表”的关系，而是“系统表背后的元信息从哪里来”的关系。
- 表级系统表本质上依赖表路径下的 Paimon 元数据文件。
- 全局系统表和 catalog 级配置、catalog 中能枚举到的表有关。
- 如果是 Hive / JDBC / REST catalog，它们管理元信息的方式不同，但 Paimon 系统表对外暴露的语义是一致的。

**一句更稳的说法**

系统表的“查询入口”是 SQL 层，底层信息来源既包括表自身元数据文件，也包括 catalog 元数据。

### 13. `sys` 是什么？

**标准回答**

`sys` 可以理解为 Paimon 暴露的系统数据库入口，用来查询 catalog 级别或全局统计类系统表。它不是普通业务库，主要服务于运维、排障和元数据分析。

### 14. 常用系统表有哪些？

**标准回答**

面试里优先记住这几个：

- `table$snapshots`：看快照历史、提交类型、manifest 引用
- `table$schemas`：看 schema 演进历史
- `table$files`：看某个 snapshot 下有哪些数据文件
- `table$manifests`：看 manifest 级元数据
- `table$options`：看表参数
- `table$consumers`：看消费位点
- `sys.catalog_options`：看 catalog 级配置
- `sys.all_table_options`：看所有表参数

---

## 六、Flink SQL、CALL、Procedure、Action Jar 高频题

### 15. `CALL xxx(...)` 是什么语法？

**标准回答**

这是 Flink SQL 的 Call Statement，用来调用 catalog 提供的 procedure。Paimon 在 Flink 1.18+ 上支持通过 SQL 直接操作部分数据和元数据，比如 compact、rollback、remove_orphan_files 等。

### 16. 为什么加了 Paimon 的 jar 包，`CALL` 就能解析？

**标准回答**

要分两层理解：

1. `CALL` 语法本身不是 Paimon 发明的，而是 Flink SQL 的能力。
2. 加上 Paimon jar 后，Paimon catalog / procedure 的实现进入了 Flink classpath，Flink 就能从 catalog 中找到对应 procedure 并执行。

所以不是“jar 让 SQL 语法出现了”，而是“jar 提供了该语法背后的 procedure 实现”。

### 17. `CALL system.rebuild_snapshots('my_table')` 应该怎么理解？

**标准回答**

`CALL` 是标准 Flink SQL procedure 调用语法；但从我检索到的官方 Paimon 文档看，官方常见写法是 `CALL sys.xxx(...)`，而不是 `system.xxx(...)`。同时，官方稳定文档里我没有检索到 `rebuild_snapshots` 是一个标准内置 procedure。

**更稳妥的判断**

- `CALL` 语法本身是标准能力。
- `system` 还是 `sys`，取决于具体 catalog / procedure 注册方式。
- `rebuild_snapshots` 很可能是你们内部扩展、某个分支能力，或者历史版本中的自定义 procedure。

这道题面试时不要把它说成“Paimon 官方固定语法”，除非你已经在代码里确认过。

### 18. 怎么给 Paimon / Flink 增加一个新的 call statement？

**标准回答**

本质不是“改 SQL 解析器”，而是“新增一个 procedure 并让 catalog 暴露它”。

可以这么答：

1. 实现一个 Flink `Procedure`。
2. 提供公开的 `call(...)` 方法作为执行入口。
3. 在 catalog 侧通过 `getProcedure` 把这个 procedure 暴露出去。
4. 把对应 jar 放进 Flink 运行时 classpath，让 SQL 层能发现并调用。

### 19. Procedure 和 Action Jar 的区别是什么？

**标准回答**

- Procedure：偏 SQL 化，适合在 Flink SQL 里直接 `CALL`。
- Action Jar：偏命令行/作业提交方式，通常通过 `flink run paimon-flink-action-xxx.jar action-name ...` 执行。

Procedure 更适合轻量运维操作；Action 更适合批量任务、离线运维、参数复杂的执行场景。

### 20. 官方有 `export-manifest` 脚本吗？

**基于本次检索的判断**

我在官方文档和公开资料里，没有检索到 `export-manifest` 作为标准内置 procedure / action 的明确信息。因此更大的可能是：

- 你们内部自定义脚本
- 某个业务工具
- 面试官其实想考的是 manifest 链路，而不是某个官方固定命令

如果面试里被问到，建议先确认它是不是“团队内部运维脚本”。如果不是，就把回答重心放在 manifest 的作用、如何从 snapshot 追到 manifest，再追到 data file。

---

## 七、运维恢复与排障高频题

### 21. Paimon 如何做回滚？

**标准回答**

核心是基于 snapshot 回滚。回滚前要确认：

- 目标 snapshot 是否完整可用
- 回滚后下游增量消费是否会受影响
- 是否需要同步处理 tag / branch / consumer 位点

### 22. Paimon 并发写会出现哪些冲突？

**标准回答**

官方文档重点提到两类：

- Snapshot conflict：别的任务先提交了新 snapshot，这类通常可以重试提交。
- Files conflict：当前作业想删的文件已被别的作业删掉，这类冲突更严重，作业可能失败并重启。

### 23. 怎么回答“Paimon 为什么适合 CDC”？

**标准回答**

因为它天然支持主键表、upsert/delete、changelog 消费、快照管理，而且能够在存储层承接 CDC 的增删改语义，不只是把 CDC 数据落成 append-only 文件。

---

## 八、你当前提纲的面试版答案

### 1. Paimon 元数据是怎么修复的？

先判断损坏层：catalog 元信息、snapshot、manifest 还是 data file；优先使用 `repair`、`rollback`、备份恢复等官方手段，避免手工篡改底层元数据文件。修复的关键不是“把某个文件补回来”，而是恢复 snapshot 到 data file 的引用一致性。

### 2. Paimon 系统表在哪，和 catalog 类型有关吗？

系统表通过 SQL 暴露，表级系统表挂在表名后缀上，比如 `$snapshots`、`$files`；全局系统表通过 `sys` 数据库访问。catalog 类型会影响元信息的存储和枚举方式，但不会改变“系统表是元数据查询入口”这个本质。

### 3. `export-manifest` 脚本

更像内部脚本名，不建议直接当成 Paimon 官方概念回答。可以把重点放在：manifest 是 snapshot 到 data file 的中间层，排障和恢复时经常需要导出或遍历 manifest 关系。

### 4. Call Statements 原理，怎么添加一个 call statement？

`CALL` 是 Flink SQL 的 procedure 调用语法。要新增一个 call statement，本质是新增 procedure 实现，并由 catalog 暴露出来，不是去改 SQL 语法本身。

### 5. `CALL system.rebuild_snapshots('my_table');` 是什么语法，为什么加了 jar 就能解析？

`CALL` 是 Flink SQL 标准语法；jar 的作用是把对应 procedure 实现和 catalog 扩展带进 classpath。至于 `system.rebuild_snapshots`，根据本次检索，它不像 Paimon 官方稳定文档里的标准内置 procedure，更像自定义扩展或历史能力。

### 6. orphan files 检测，怎么清除？

本质是找“不再被任何 snapshot 引用”的文件。官方做法是 `remove_orphan_files`，生产上建议先 dry run，再删除，并确认没有仍在写入但尚未稳定提交的任务。

---

## 九、面试速记版

如果你只能记 6 句话，优先记这几句：

1. Paimon 适合 CDC、主键更新和实时湖仓场景。
2. 主键表底层是 LSM，读性能很依赖 compaction。
3. 元数据主链路是 `snapshot -> manifest list -> manifest -> data file`。
4. 系统表是查询元数据的入口，既有 `table$xxx`，也有 `sys.xxx`。
5. `CALL` 是 Flink SQL procedure 语法，Paimon jar 提供的是实现，不是语法本身。
6. 元数据恢复的核心是“恢复引用一致性”，不是手工修单个文件。

---

## 十、资料来源

以下结论主要基于官方资料整理：

- Paimon Basic Concepts: https://paimon.apache.org/docs/master/concepts/basic-concepts/
- Paimon System Tables: https://paimon.apache.org/docs/0.9/maintenance/system-tables/
- Paimon Flink Procedures: https://paimon.apache.org/docs/1.1/flink/procedures/
- Paimon Manage Snapshots: https://paimon.apache.org/docs/1.1/maintenance/manage-snapshots/
- Paimon Concurrency Control: https://paimon.apache.org/docs/1.0/concepts/concurrency-control/
- Paimon API - Procedure classes: https://paimon.apache.org/docs/1.1/api/java/org/apache/paimon/flink/procedure/CompactProcedure.html
- Flink Procedure API: https://nightlies.apache.org/flink/flink-docs-master/api/java/org/apache/flink/table/procedures/Procedure.html

说明：

- 关于 `export-manifest`、`rebuild_snapshots`，本次未在官方稳定文档中检索到明确的标准内置能力，因此文中将其判断为“更可能是自定义扩展/内部脚本”。这是基于公开资料的推断，不是官方明文说明。
