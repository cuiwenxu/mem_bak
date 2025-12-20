
# Paimon 中 `changelog-producer` 的几种选项

根据 Paimon 官方文档，可配置的值有：`none`、`input`、`lookup`、`full-compaction`。 ([Apache Paimon][1])

具体说明如下：

| 值                   | 含义                                                            | 适用场景 /优缺点                                                                                                                                                                                                                                  |
| ------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **none**            | 不生成额外的 changelog 文件。                                          | 默认模式。写入表只合并 snapshot，不保留逐条变更。适合 downstream 只关心最新状态、不需要 “update_before” 的场景。 ([Apache Paimon][2]) 这种方式性能开销最低。                                                                                                                               |
| **input**           | 将写入的输入视作完整的 changelog，Paimon 会把输入的每条记录写进 changelog 文件。        | 当输入本身就是完整的变更日志（比如来自 MySQL CDC）时非常合适。无需额外的计算开销。 ([Apache Paimon][2]) 但如果输入不是完整变化，就可能不准确。                                                                                                                                                    |
| **lookup**          | 使用 “lookup” 机制在提交数据前生成 changelog。Paimon 会先查出之前老值，然后和新值比对生成变化。 | 适合输入不是完整 changelog，但你仍需要完整变更流（包括 update_before、update_after）。比 full-compaction 延迟低，但会消耗较多资源。 ([Apache Paimon][2]) 支持 `lookup.cache-max-memory-size` 和 `lookup.cache-max-disk-size` 等配置来控制资源。 ([Apache Paimon][2])                          |
| **full-compaction** | 每次 **全 compaction** 后生成 changelog，通过比较 compaction 前后的结果来计算变更。 | 适合对变更延迟要求不高（例如小时级）的场景。资源开销较小（因为是 compaction 本身做对比），但延迟高。 ([Apache Paimon][2]) 可以配置 `full-compaction.delta-commits` 来决定多少次 commit 后做一次 full compaction。 ([Apache Paimon][2]) 同样支持 `row-deduplicate` 来去除不必要的 `-U`/`+U`。 ([Apache Paimon][3]) |

---

## ⚙️ 推荐建议（选哪个）

* 如果你是 **Flink CDC（MySQL binlog）写入 Paimon**：推荐使用 `input`。
* 如果你的输入不是完整变更，但你需要 downstream 完整变更流：推荐 `lookup`。
* 如果你对实时性要求不高，只想定期输出日志变更（做日报/小时报）：可以考虑 `full-compaction`。
* 如果只是做 snapshot 查询，不做持续流处理：可以用 `none`。

# 生成时机与快照行为


- NONE ：不生成变更日志文件，增量流读只依赖数据清单（base/delta）计算变化

- INPUT ：在内存表 flush（追加提交）时双写变更日志，变更源自输入
```java
private void flushWriteBuffer(boolean waitForLatestCompaction, boolean forcedFullCompaction)
            throws Exception {
        if (writeBuffer.size() > 0) {
            if (compactManager.shouldWaitForLatestCompaction()) {
                waitForLatestCompaction = true;
            }

            // 如果changelog type=input,则新建一个changelog writer，双写
            // 否则不生成额外的writer
            final RollingFileWriter<KeyValue, DataFileMeta> changelogWriter =
                    changelogProducer == ChangelogProducer.INPUT
                            ? writerFactory.createRollingChangelogFileWriter(0)
                            : null;
            final RollingFileWriter<KeyValue, DataFileMeta> dataWriter =
                    writerFactory.createRollingMergeTreeFileWriter(0, FileSource.APPEND);

            ...
        }
}
```

- FULL_COMPACTION ：每次全量压缩时生成变更日志，变更源自压缩对现有数据的合并与重写

- LOOKUP ：提交数据前通过对存储做查找生成变更日志，变更基于“写前对比”


# input模式下，写入的文件存储在哪，格式是什么？



- 快照内的清单分层：
  - 数据清单列表： baseManifestList 、 deltaManifestList （全量+本次增量），见 paimon-api/src/main/java/org/apache/paimon/Snapshot.java:314-334
  - 变更日志清单列表： changelogManifestList （仅在启用变更日志时存在），见 paimon-api/src/main/java/org/apache/paimon/Snapshot.java:336-346
- 读取方式：
  - 数据清单： ManifestList.readDataManifests(snapshot) / readDeltaManifests(snapshot) ，见 paimon-core/src/main/java/org/apache/paimon/manifest/ManifestList.java:88-102
  - 变更日志清单： ManifestList.readChangelogManifests(snapshot) ，见 paimon-core/src/main/java/org/apache/paimon/manifest/ManifestList.java:109-113
- 历史差异（0.2）：早期 INPUT 可能把变更文件挂在数据文件的 extraFiles ，后续迁移到增量结构，注释见 paimon-core/src/main/java/org/apache/paimon/io/DataFileMeta.java:337-345


# 结合源码分析changeLog模式的异同
- input模式会把输入的每条记录都写到changelog文件中
