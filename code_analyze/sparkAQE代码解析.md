# spark AQE代码解析
AQE 既定的规则和策略主要有4个，分为1个逻辑优化规则和3个物理优化策略。我把这些规则与策略，和相应的 AQE 特性，以及每个特性仰仗的统计信息，都汇总到了如下的表格中。
| 优化类型 | 规则与策略 | AQE特性 | 统计信息 |
| --- | --- | --- | --- |
| 逻辑计划 | DemoteBroadcastHashJoin | join策略调整 | map阶段中间文件 |
| 物理计划 | OptimizeLocalShuffleReader | join策略调整 | map阶段中间文件 |
| 物理计划 | CoalesceShufflePartitions | 自动分区合并 | reduce task分区大小 |
| 物理计划 | OptimizedSkewedJoin | 自动倾斜处理 | reduce task分区大小 |

## BHJ降级解析
```java
 private def selectJoinStrategy(
      join: Join,
      isLeft: Boolean): Option[JoinStrategyHint] = {
    val plan = if (isLeft) join.left else join.right
    plan match {
      case LogicalQueryStage(_, stage: ShuffleQueryStageExec) if stage.isMaterialized
        && stage.mapStats.isDefined =>
        //分析空分区占比
        val manyEmptyInPlan = hasManyEmptyPartitions(stage.mapStats.get)
        //根据join类型判断是否可广播，比如是否等值连接
        val canBroadcastPlan = (isLeft && canBuildBroadcastLeft(join.joinType)) ||
          (!isLeft && canBuildBroadcastRight(join.joinType))
        val manyEmptyInOther = (if (isLeft) join.right else join.left) match {
          case LogicalQueryStage(_, stage: ShuffleQueryStageExec) if stage.isMaterialized
            && stage.mapStats.isDefined => hasManyEmptyPartitions(stage.mapStats.get)
          case _ => false
        }

        val demoteBroadcastHash = if (manyEmptyInPlan && canBroadcastPlan) {
          join.joinType match {
            // don't demote BHJ since you cannot short circuit local join if inner (null-filled)
            // side is empty
            case LeftOuter | RightOuter | LeftAnti => false
            case _ => true
          }
        } else if (manyEmptyInOther && canBroadcastPlan) {
          // for example, LOJ, !isLeft but it's the LHS that has many empty partitions if we
          // proceed with shuffle.  But if we proceed with BHJ, the OptimizeShuffleWithLocalRead
          // will assemble partitions as they were before the shuffle and that may no longer have
          // many empty partitions and thus cannot short-circuit local join
          join.joinType match {
            case LeftOuter | RightOuter | LeftAnti => true
            case _ => false
          }
        } else {
          false
        }
```
在它们分别完成 Shuffle Map 阶段的计算之后，DemoteBroadcastHashJoin 会判断中间文件是否满足如下条件：
• 中间文件尺寸总和小于广播阈值 spark.sql.autoBroadcastJoinThreshold
• 空文件占比小于配置项 spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin