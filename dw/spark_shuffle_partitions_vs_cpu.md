## spark.sql.shuffle.partitions 与集群 CPU 的关系

### 1. 基本概念

- **总 CPU 核数 C** ≈ `numExecutors * executorCores`（假设 `spark.task.cpus=1`）。
- **spark.sql.shuffle.partitions** 决定 **Shuffle 之后 RDD 的分区数 ≈ Task 数**。
- 目标：**Task 数与 CPU 核数同一量级**，既跑满 CPU，又不过度切碎成大量短 Task。

---

### 2. 经验原则

- 理想情况：
  - `spark.sql.shuffle.partitions ≈ 1～3 * C`
  - 即：Shuffle 后的 Task 数大于等于核数，但不要大太多。

- 调参示例：
  - 集群：20 executors * 4 cores = 80 cores → `spark.sql.shuffle.partitions = 80 ~ 240`。
  - 小集群：8～16 核 → 建议 `spark.sql.shuffle.partitions = 16 ~ 32`，不要用默认 200。

---

### 3. 如何根据表现微调？

- **看 Task 时长**：
  - 大部分 Task < 10 秒 → 分区偏多，可以适当减小 partitions。
  - 大部分 Task > 2 分钟 → 分区偏少，可以适当增大 partitions。

- **看 CPU 利用率**：
  - CPU 长时间 < 50%，且 Task 总数 < C → 分区偏少，增加 partitions。
  - CPU 很高但 Task 特别多且很短 → 分区偏多，减小 partitions。

调参策略：
1. 先按 `C ~ 2C` 设一个初值；
2. 跑一两次，看 Spark UI 中各 stage 的 Task 时长和数量；
3. 过短就减半，过长就乘 1.5～2，逐步收敛。

---

### 4. 与 AQE 的关系

在 Spark 3 开启 AQE：

```sql
set spark.sql.adaptive.enabled=true;
set spark.sql.adaptive.coalescePartitions.enabled=true;
```

- `spark.sql.shuffle.partitions` 作为 **初始上限**；
- AQE 会根据真实数据量自动 **合并过小 partition**；
- 仍建议初始值与 `CPU 核数同级`，让 AQE 在合理区间内微调，而不是从 200 这种严重偏大的默认值开始。
