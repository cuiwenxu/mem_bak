# flink内存管理
JVM 存在的几个问题：

- Java 对象存储密度低。一个只包含 boolean 属性的对象占用了16个字节内存：对象头占了8个，boolean 属性占了1个，对齐填充占了7个。而实际上只需要一个bit（1/8字节）就够了。
- Full GC 会极大地影响性能，尤其是为了处理更大数据而开了很大内存空间的JVM来说，GC 会达到秒级甚至分钟级。
- OOM 问题影响稳定性。OutOfMemoryError是分布式计算框架经常会遇到的问题，当JVM中所有对象大小超过分配给JVM的内存大小时，就会发生OutOfMemoryError错误，导致JVM崩溃，分布式框架的健壮性和性能都会受到影响。

## flink内存管理针对实时场景做了哪些优化

积极的堆外内存管理，因为流处理注重低延迟，为每一条要处理的数据在堆内存上创建对象会触发频繁的gc，导致处理停滞
- flink在堆外开辟network buffer（专门存放缓冲数据）
- 开辟managed memory，用于排序，缓存中间值 

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2d7397a5590f17b9f6bd72d272d369f2.png)


# 怎么定位背压
flink拓扑图，Flink的下游算子无法及时处理上游的消息时会出现反压的提示。反压提示一般有OK,LOW,HIGH三种状态。某个算子的BackPressure指标如果是HIGH，说明后面的算子存在性能问题。在任务性能调优时，对于串在一起的算子(Flink会对并发度一样，在同一个slotgroup下，且允许和其他链在一起的算子进行串联)可以disableChaining，然后分析各个算子的性能。
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1a9627bac0f30bf29ac87cd322acb6c3.png)
ratio计算原理

# 怎么处理背压
1. 增加并行度
增加下游任务的并行度可以帮助提高处理速度，从而减轻背压。这可以通过配置更多的 Task Slots 或调整并行度参数来实现。

2. 调整缓冲区大小
Flink 允许调整网络缓冲区的大小。通过增加缓冲区大小，可以为下游任务提供更多的缓冲空间来处理突发的数据。然而，这只是临时缓解背压的方法，并不能从根本上解决问题。

3. 优化代码
检查下游任务的代码，看是否有优化空间。有时候，算子的代码可能存在性能瓶颈，如不必要的数据转换、低效的算法或过多的状态访问。优化这些代码可以提高处理速度。
写redis,mget和mset，要

4. 使用异步 I/O
对于涉及外部系统调用的操作，使用异步 I/O 可以避免阻塞任务的执行线程。Flink 提供了异步 I/O API，允许你以异步方式执行数据库查询、HTTP 请求等操作。

5. 调整 Checkpoint 配置
首先说明checkpoint和反压的关系，checkpoint分为对齐和非对齐，对齐的checkpoint会阻断快流的消费，非对齐的checkpoint是把快流数据缓存下来，虽不会阻断消费，但是会占用内存空间
所以，checkpoint的配置策略应该是
```
1.业务场景下允许的情况下，最好使用AT_LEAST_ONCE而非EXACTLY_ONCE
2.EXACTLY_ONCE语义下，可以启用非对齐的checkpoint，env.getCheckpointConfig().enableUnalignedCheckpoints()。
3.可以设置两次checkpoint之间的时间间隔，尤其在作业状态较大或任务反压时效果比较明显 env.getCheckpointConfig().setMinPauseBetweenCheckpoints()
```

# AT_LEAST_ONCE和EXACTLY_ONCE性能差异在哪
## 至少一次（AT_LEAST_ONCE）
在至少一次的语义下，Flink 保证在发生故障时，所有的记录至少被处理一次。这意味着在某些情况下，消息可能会被处理多次，从而可能导致数据重复。
性能方面：
资源使用：通常，至少一次语义需要较少的资源***（只需要记录Kafka消费位点即可）***，因为它不需要在每次记录处理时进行严格的状态检查和记录。
延迟：至少一次语义可能会导致较低的处理延迟，因为它不需要等待复杂的容错机制。
吞吐量：可能会有较高的吞吐量，因为系统不需要频繁地进行状态快照和同步。

## 精确一次（EXACTLY_ONCE）
精确一次语义保证每条记录在整个数据流中只被处理一次，即使在发生故障的情况下。为了实现这一点，Flink 需要定期进行状态快照，并确保状态的一致性。
性能方面：
资源使用：精确一次语义通常需要更多的资源，特别是在状态管理和检查点（Checkpoint）机制上。
延迟：处理延迟可能会更高，因为系统需要等待状态快照完成，并且可能需要进行额外的同步操作。
吞吐量：吞吐量可能会受到影响，因为系统需要定期执行检查点操作，这可能会暂时阻塞数据的处理。

# 怎么优化存储空间
## 优化难易程度：

分区生命周期 < 存储格式优化 < 视图化 < string2int，类型最小化 < 行转列，复合数据类型压缩，bitmap行压缩 < 大文本范式化 < 无用字段剪枝 < 快照转拉链

## 优化效果：

视图化 > 分区生命周期优化，无用字段剪枝 > 存储格式优化 > 行转列，复合数据类型压缩，bitmap行压缩 > 大文本范式化 > string2int，类型最小化 > 快照转拉链

得分低，代表推荐优先级高，因此通用推荐优化顺序为
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/500981289c74c81377822e11e715ff76.png)
orc格式存储
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e140a646f1829d2c30d79e4b3b96d827.png)
对于使用orc格式存储的表，可以对列进行排序
- 1.选取order by 字段，此处使用字段关联性分析
>原理是根据相同相似增加压缩率的原理
>即该字段有序后，其他字段也相对有序，比如有uuid、order_id、client_type、order_ts
>此时order by字段应该是uuid
- 2.order by 关联性最高的字段，起批任务，定期回刷历史分区

# flink背压数据怎么计算


# flink读写redis的性能优化
背景：flink任务高峰期背压大，checkpoint失败（barrier对不齐，导致checkpoint超时）
排查方式：根据flink拓扑图，背压是写kv这一步导致，首先尝试提高flink并发度，发现写入耗时降不下来，并且kv数据库运维反馈集群连接数占用太高
基于此，去翻看redis客户端代码，发现mget/mset操作请求时，会遍历这一批数据的key，获取他们所在的bucket，然后对不同的bucket进行异步请求，最坏的情况下，8条数据分别都落在Cellar中的8个节点上，其实质还是异步的8次批量操作分别请求了16个节点，每个批量请求其实还是还是一条数据。
```java
redis client一致性hash代码，取murmurhash，然后对bucketCount取模
protected int findServerIndex(byte[] prefix, byte[] index) {
		ChannelBuffer buffer = null;
		if (prefix != null) {
			buffer = ChannelBuffers.wrappedBuffer(prefix, index, null);
		} else {
			buffer = ChannelBuffers.wrappedBuffer(index, null);
		}
		long hash = TairUtil.murMurHash(buffer);
		if ((dsList != null) && (dsList.size() > 0))
			return (int) (hash %= bucketCount);
		else {
			maybeForceCheckVersion();
			return -1;
		}
	}

```
参照redis数据分布的策略，提高mget/mset缓存无底洞的效率

# flink性能优化——roaringbitmap去重代替set去重
# flink性能优化——




# Catalyst逻辑优化
- unresolved logical plan (只进行语法树解析)
- analyzed logical plan，结合表schema，得到字段数据类型
- optimized logical plan，从以三个方向的优化
   1.谓词下推（Predicate Pushdown）,谓词”指代的是像用户表上“age < 30”这样的过滤条件，“下推”指代的是把这些谓词沿着执行计划向下，推到离数据源最近的地方，从而在源头就减少数据扫描量。换句话说，让这些谓词越接近数据源越好
   2.列剪裁（Column Pruning）列剪裁就是扫描数据源的时候，只读取那些与查询相关的字段
   3.常量替换 （Constant Folding）

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/2f0fb190e8c9305af60517bebcb64d67.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/081ac6aafc25951ecc1e40f8721c0b19.png)
# lsm原理
写入优化
1. 顺序写入，数据先写入内存缓冲区（MemTable），批量刷盘生成有序文件（SSTable），避免磁盘随机I/O
2. 冷热数据分级​​：新数据存储在高速内存层，旧数据逐步下沉到低速磁盘层，利用存储介质特性提升效率

读取优化
分层查询：读操作需依次检查内存层（MemTable）和磁盘多层SSTable，通过​​布隆过滤器​​快速跳过无效文件，减少磁盘扫描

# paimon的合并引擎
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9278bda2e20f4f369258a077ee228195.png)

# paimon

# watermark作用
## 1.事件时间的推进
上游发过来的数据总是乱序的，有早有晚，然而有些动作是必须要明确的标记触发的，比如窗口计算。
那么，此时就需要一个水位线来推进事件时间
比如，系统最大时间-时间间隔
```java
stream.assignTimestampsAndWatermarks(
  WatermarkStrategy
    .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(10))
    .withTimestampAssigner((event, ts) -> event.getEventTime())
);
```
表示系统允许最大乱序为 10 秒。

- Flink 在内部生成 Watermark 的公式为：Watermark = 当前观察到的最大事件时间 - 10 秒

- 只要比这个 Watermark 更早的事件，就被认为是“已经延迟太久”的数据，可能被丢弃或作为迟到数据处理。
## 2.多流操作的协同​
​​双流 Join 的完整性保证​​：在双流 join或 coProcessFunction中，Watermark 用于对齐两个流的事件时间。例如：
- 流 A 的 Watermark = 10:05
- 流 B 的 Watermark = 10:00

下游算子以 min(10:05, 10:00) = 10:00作为当前 Watermark，确保仅当两流数据均推进到 10:00后才输出匹配结果，避免数据丢失 。
​​乱序数据处理​​：通过取各输入流的最小 Watermark，确保慢速流的数据不被快速流覆盖

## 3. 状态管理优化​
- 状态自动清理​​：Watermark 可用于触发状态过期清理（State TTL）。例如，设置状态保留时间为 1 小时，当 Watermark 超过某状态的“最大有效时间戳 + 1 小时”时，自动清理该状态，避免内存无限增长
​​原理​​：基于数据的事件时间戳（Event Timestamp）和 Watermark 判断过期。状态失效时间 = ​​事件时间戳 + TTL​​，需依赖 Watermark 推进全局事件时间。
​​适用场景​​：需按业务时间精确清理状态的场景（如会话超时、跨天统计）。
​​配置方法​​：
```java
StateTtlConfig ttlConfig = StateTtlConfig.newBuilder(Time.hours(6))
    .setTtlTimeCharacteristic(TtlTimeCharacteristic.EventTime)
    .build();
```

# 撤回流和覆盖写的场景和优缺点
数据修正能力：多版本回溯 vs 单版本覆盖​
- 撤回流​​
	- ​机制​​：通过二元组 (Boolean, Row) 表达变更，false 标记撤回旧值，true 标记新增值。
		
			示例：聚合统计中，若某商品销售额原为 100元（已下发），新数据到达后更新为 150元，则先发 (false, 100) 撤回旧值，再发 (true, 150) 更新结果。
	- ​​优势​​：
		- ​支持历史状态修正​​：可处理迟到数据（Late Data）或计算错误，确保最终结果严格匹配所有事件时序。
		- ​避免覆盖丢失​​：旧值被显式撤回，下游可追溯完整变更链，适用于金融风控、实时审计等场景。

- ​​覆盖写​​
	- 机制​​：直接写入新值覆盖旧值（如 Redis 的 SET key 150）。
	- ​缺陷​​：
		- 历史状态丢失​​：旧值被永久覆盖，无法回溯中间状态（如无法得知 100→150 的变更过程）。
		- 无法修正错误​​：若因乱序需修正结果，覆盖写只能依赖最新值，导致累计误差