## 理解 Spark Physical Plan：从一个真实拉链案例说起

这篇文章基于一份真实的 Spark 执行计划和对应的 SQL/脚本，整理成一篇面向实战的笔记，重点回答几个常见问题：

- **Physical Plan 怎么读，从哪入手？**
- **`(45)` 这种节点序号到底是什么意思？**
- **为什么 SQL 里有 `where dp = 'active'`，Plan 里却看不到 `Filter dp='active'`？**
- **`PartitionFilters` 是什么，为什么不在树形结构里？**
- **Spark 常见物理算子和关键参数有哪些？**

---

## 一、先看“骨架”：这条 SQL 在干嘛？

示例脚本（简化）大致逻辑如下（来自 `dwd_clt_en_hwycconsume_b_recharge_h_chain_test.sh`）：

- 建表：`dwd.dwd_clt_en_hwycconsume_b_recharge_h_chain`
- **缓存 4 个中间表**：
  - `historical_data`：`dp = 'active'` 的历史拉链
  - `incremental_data`：当天增量（从 `dp='increment'` 分区读）
  - `merged_data`：`historical_data UNION ALL incremental_data`，并加上 `source_priority` 等字段
  - `latest_records`：在 `merged_data` 上做窗口，做去重和状态计算
- **最终一次 `insert overwrite`** 写入测试表：
  - 一支：`expired` 分区（历史被新数据覆盖的老版本）
  - 一支：`active` 分区（当前最新版本）

在 Physical Plan 的最外层，可以看到类似结构：

- `AdaptiveSparkPlan`
  - `Execute InsertIntoHadoopFsRelationCommand`
    - `WriteFiles`
      - `Sort`
        - `Exchange`
          - `Union`（expired 分支 + active 分支）

**阅读 Physical Plan 的推荐顺序：**

1. **从最外层算子开始**：看它在干什么（这里是 `InsertIntoHadoopFsRelationCommand` → 写 ORC 分区表）。
2. **往下看 Union 两支**：分别对应 expired / active 逻辑。
3. **再追溯到数据源**：`Scan orc ...`，看从哪些分区/路径读数据。
4. **中间重点盯 Window / Exchange / Sort / Cache**：这些通常是性能热点。

---

## 二、节点序号 `(45)` 是什么？和结果顺序有关系吗？

Plan 中经常看到类似：

```text
+- Exchange (45)
...
(45) Exchange
Input [25]: ...
Arguments: hashpartitioning(dp#..., dt#..., end_date#..., 200), ...
```

### 1. 序号的含义

- 括号里的数字（如 `(45)`）是 **Spark 给该物理算子在这份计划内分配的唯一编号**。
- 作用：
  - 在“树形结构”中：标记每个算子。
  - 在“下方详细信息区”中：用 `(45)` 开头来给出这个算子的 **Input、Arguments、Condition 等详细参数**。
- 本身**没有业务含义**，也不是 Stage ID / Task ID。

### 2. 数字大小能不能理解为“越大越靠近结果”？

- 一般情况下：Spark 打印计划时会先给下面的子节点编号，再给上面的父节点，因此 **数字通常越大越靠近根（最终结果）**。
- 但要注意：
  - 这属于实现细节的“遍历顺序”，**不能拿来做任何语义判断**（比如代价、耗时、Stage 顺序等）。
  - 遇到 Adaptive Plan、子查询、多阶段时顺序可能更“跳”。

**可以记为**：  
`(n)` 只是“当前执行计划内部的节点索引”，方便你在树和详细区之间对号入座——**只当索引用，不当语义用**。

---

## 三、`where dp = 'active'` 去哪了？为什么看不到 Filter？

脚本里的一个关键 SQL 片段（历史数据缓存）是：

```sql
CACHE LAZY TABLE historical_data as
select *
from dwd.dwd_clt_en_hwycconsume_b_recharge_h_chain
where dp = 'active';
```

在 Physical Plan 中，对应的底层 Scan 片段（概念上类似）是：

```text
Scan orc spark_catalog.dwd.dwd_clt_en_hwycconsume_b_recharge_h_chain
Location: .../dwd_clt_en_hwycconsume_b_recharge_h_chain/dp=active/...
PartitionFilters: [isnotnull(dp#...), (dp#... = active)]
DataFilters: []
...
InMemoryRelation(...)  -- historical_data
```

### 1. 为何没有单独的 `Filter(dp='active')` 算子？

因为：

- `dp` 是 **分区列**。
- Spark/Hive 在处理 `where dp = 'active'` 时，会把它变成 **分区裁剪**：
  - 只扫描 `dp=active` 这个分区目录。
- 在物理计划里，这个优化体现在 **Scan 节点的参数上**，而不是单独的 `Filter` 算子，所以你看到的是：

- `PartitionFilters: (dp = active)`  
- 而不是树里某一行 `Filter (dp = active)`。

### 2. 如何确认条件生效？

看 Scan 节点下的这几个字段：

- **`PartitionFilters`**：对分区列的过滤条件（是否只扫了需要的分区）。
- **`DataFilters`**：对非分区列的过滤条件。
- **`PushedFilters`**：真正下推到存储格式（如 ORC/Parquet）的过滤。

**结论：**

- SQL 里 `where dp = 'active'` 是生效的，只不过：
  - **对分区列的过滤被合并进 Scan 节点，展示在 `PartitionFilters` 中**；
  - 树形结构只展示“算子类型关系”，不展示这些内部参数。

---

## 四、为什么树形结构中看不到 `PartitionFilters`？

Physical Plan 输出一般分成两部分：

1. **上半部分：树形结构（结构图）**

   例如：

   ```text
   * ColumnarToRow (8)
     +- Scan orc spark_catalog.dwd.dwd_clt_en_hwycconsume_b_recharge_h_chain (7)
   ```

   - 只告诉你：这里有一个 `Scan orc` 节点，下面接了一个 `ColumnarToRow`。
   - 重点是 **算子之间的父子关系**，而不是参数细节。

2. **下半部分：节点详细信息**

   例如：

   ```text
   (7) Scan orc spark_catalog.dwd.dwd_clt_en_hwycconsume_b_recharge_h_chain
   Output [25]: ...
   Location: ...
   PartitionFilters: [isnotnull(dp#...), (dp#... = active)]
   ReadSchema: struct<...>
   ```

   - 这里才是 Scan 的 **参数详情**：读取路径、分区过滤、读取 Schema 等。

**因此：**

- `PartitionFilters` 是 **Scan 节点内部的一个属性**，不会在树形结构里升格成一个单独算子。
- 树形结构中只会有单独 `Filter` 节点，针对的是：
  - 非分区列过滤，或
  - 无法下推的复杂条件。

---

## 五、示例逻辑拆分：这几个算子是怎么配合工作的？

结合这份拉链逻辑，可以从物理计划中划出几个关键模块，方便以后看类似的计划。

### 1. 增量去重（`incremental_data`）

SQL 中：

```sql
select
  ...
  ,getmd5(...) as change_code
  ,row_number() over (distribute by id,user_id sort by id,user_id,binlog_ts desc) as keyrank
from ...
where dp='increment' and dt between ...
  and id is not null and user_id is not null
...
where keyrank = 1;
```

Physical Plan 中对应关键算子：

- `Scan orc ... dp=increment, dt in 范围` → `PartitionFilters`
- `Filter (isnotnull(id) AND isnotnull(user_id))`
- `Project (生成 change_code)`
- `Exchange(hashpartitioning(id,user_id,...))`
- `Sort(order by id,user_id,binlog_ts desc)`
- `Window(row_number over(...))`
- `Filter (keyrank = 1)`
- `Project` → `InMemoryRelation[incremental_data]`

**这是一个典型“增量按主键取最新”的 Window 模式。**

### 2. 历史 + 增量合并（`merged_data`）

SQL 中：

```sql
CACHE LAZY TABLE merged_data as
  select ..., 1 as source_priority, 'history' as source_type from historical_data
  union all
  select ..., 2 as source_priority, 'incremental' as source_type from incremental_data;
```

Physical Plan 中：

- `Scan In-memory table historical_data` → `Project(source_priority=1, source_type='history')`
- `Scan In-memory table incremental_data` → `Project(source_priority=2, source_type='incremental')`
- `Union`
- `InMemoryRelation[merged_data]`

### 3. 计算最新有效记录（`latest_records`）

SQL 中：

```sql
CACHE LAZY TABLE latest_records as
select
  ...
  ,source_priority
  ,source_type_sum
  ,source_rn_data
from (
  select
    *
    ,row_number() over (partition by id,user_id order by source_priority desc) as source_rn_data
    ,sum(source_priority) over (partition by id,user_id) as source_type_sum
  from merged_data
) ranked;
```

Physical Plan 中：

- `Scan In-memory table merged_data`
- `Exchange(hashpartitioning(id,user_id))`
- `Sort(id,user_id,source_priority desc)`
- `Window(row_number ...)` → `source_rn_data`
- `Window(sum(source_priority) ...)` → `source_type_sum`
- `Project` → `InMemoryRelation[latest_records]`

### 4. 拆成 expired / active 两支写出

SQL 中：

```sql
-- expired
select ..., 'expired' dp, '${data_day}' dt, '${data_day}' end_date
from latest_records
where source_type_sum = 3 and source_rn_data = 2

union all

-- active
select ..., 'active' dp, '99991231' dt, '99991231' end_date
from latest_records
where source_rn_data = 1
```

Physical Plan 中：

- expired 分支：
  - `Scan In-memory table latest_records`
  - `Filter(source_type_sum = 3 AND source_rn_data = 2)`
  - `Project(dp='expired', dt=data_day, end_date=data_day)`
- active 分支：
  - `Scan In-memory table latest_records`
  - `Filter(source_rn_data = 1)`
  - `Project(dp='active', dt=99991231, end_date=99991231)`
- `Union`
- `Exchange(hashpartitioning(dp,dt,end_date))`
- `Sort(dp,dt,end_date)`
- `WriteFiles` → `InsertIntoHadoopFsRelationCommand`

---

## 六、Spark 常见物理算子和关键参数速查表

### 1. 常见物理算子

- **扫描类**
  - `Scan orc/parquet/json/csv`
  - `FileScan`
  - `BatchScan`
  - `InMemoryRelation`（缓存表）

- **基本算子**
  - `Filter`
  - `Project`
  - `Aggregate` / `HashAggregate` / `SortAggregate`
  - `Window`
  - `Sort`
  - `Union`
  - `Limit` / `TakeOrderedAndProject`

- **Join / Shuffle**
  - `BroadcastHashJoin`
  - `ShuffledHashJoin`
  - `SortMergeJoin`
  - `Exchange`（所有 Shuffle 的根）
  - `Repartition` / `RepartitionByRange`
  - `Coalesce`

- **行列转换 / Codegen**
  - `ColumnarToRow`
  - `RowToColumnar`
  - `WholeStageCodegen`

- **写入 / 命令**
  - `WriteFiles`
  - `InsertIntoHadoopFsRelationCommand`
  - 各种 `*Command`（建表、改表等）

- **AQE 相关**
  - `AdaptiveSparkPlan`
  - `ShuffleQueryStage`
  - `BroadcastQueryStage`

### 2. 关键参数字段

看 Physical Plan 时，优先关注这些字段：

- **Scan**
  - `Location`：路径/分区目录
  - `PartitionFilters`：按分区列过滤
  - `DataFilters`：按数据列过滤
  - `PushedFilters`：真正下推到存储格式的过滤
  - `ReadSchema`：读取的列

- **Filter**
  - `Condition`：完整条件表达式

- **Project**
  - `Output`：输出列和表达式
  - `Input`：输入列

- **Exchange**
  - `Arguments`：`hashpartitioning(keys, numPartitions)` 等，决定 Shuffle 行为

- **Sort**
  - `Arguments`：排序键和顺序

- **Window**
  - `Arguments`：`row_number()` / `sum()` 等 window 函数定义，含 partition/order/frame

- **WriteFiles / Insert**
  - 输出路径、分区列、文件格式、写入模式 (`Overwrite` / `Append`)

---

## 七、实战阅读 Spark Physical Plan 的推荐流程

结合本文内容，可以把“看一份 Physical Plan”的流程固化为下面几个步骤：

1. **从最外层开始**  
   找到 `AdaptiveSparkPlan → Execute InsertInto...`，明确：
   - 写入哪张表、哪种格式？
   - 分区列是哪些？
   - 写入模式是 Overwrite 还是 Insert Into？

2. **拆分主干逻辑**  
   看 `Union`、`Join`、`Window` 等大的逻辑块，把整条 SQL 在脑中分成几段功能（如：增量去重、历史合并、状态拆分等）。

3. **回溯数据源**  
   找到所有 `Scan orc/parquet`：
   - 看 `Location` 是否只扫了你想要的路径。
   - 看 `PartitionFilters` / `DataFilters` 是否正确反映 SQL 条件。
   - 看 `ReadSchema` 是否有做列裁剪。

4. **锁定性能热点**  
   特别盯：
   - `Exchange`（Shuffle）的位置和分区键。
   - `Sort` 和 `Window` 所在的位置和输入数据量。
   - `InMemoryRelation` 的使用是否合理（是否缓存了足够复用、又不会内存爆炸的中间结果）。

5. **对照 SQL 做逻辑验证**  
   - where 分区列 → 对照 `PartitionFilters`。
   - where 非分区列 → 对照 `Filter.Condition`。
   - `row_number/sum over` → 对照 `Window.Arguments`。
   - `insert overwrite partition (...)` → 对照最终 `WriteFiles` 的分区列和路径。

做到这几点之后，你再看类似的 Physical Plan，就不会只看到一堆“乱码”，而是能快速定位：**它在做什么、哪里可能出问题、要从哪里优化入手**。
