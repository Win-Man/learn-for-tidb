| Author | Reviewer | Version | Update Time | link |
| ------ | -------- | ------- | ----------- | ---- |
| 沈刚 | - | v1.0 | 2021-03-08 | - |

# 统计信息

## 统计信息介绍

在基于代价规则的物理优化阶段，优化器会基于统计信息和不同的属性选择等生成各种不同代价的物理计划，通过比较物理计划的代价，最后选择一个代价最小的物理计划。因为一般 CBO 都是根据统计信息和代价模型计算每个执行计划的 Cost ，从中选择 Cost 最小的执行计划。如果统计信息计算的不准确可能会导致最终生成的执行计划并不是代价最小、最优的执行计划。

在 TiDB 中统计信息收集了表相关以及列相关的信息，表的统计信息包括总行数和修改的行数。列的统计信息包括不同值的数量、NULL 的数量、直方图、列上出现次数最多的值 TOPN 以及该列的 Count-Min Sketch 信息。

下面介绍两种 TiDB 中的统计信息方式。

### 直方图

直方图是一种对数据分布情况进行描述的工具，它会按照数据的值大小进行分桶，并用一些简单的数据来描述每个桶，比如落在桶里的值的个数。通过直方图可以比较容易估算区间查询的代价。根据分桶策略的不同，常见的直方图可以分为等深直方图和等宽直方图。在 TiDB 中，会对每个表具体的列构建一个等深直方图，区间查询的估算便是借助该直方图来进行。

等深直方图，就是让落入每个桶里的值数量尽量相等。举个例子，比方说对于给定的集合 {1.6, 1.9, 1.9, 2.0, 2.4, 2.6, 2.7, 2.7, 2.8, 2.9, 3.4, 3.5}，并且生成 4 个桶，那么最终的等深直方图就会如下图所示，包含四个桶 [1.6, 1.9]，[2.0, 2.6]，[2.7, 2.8]，[2.9, 3.5]，其桶深均为 3。

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20210308153910.png)

### Count-Min Sketch

Count-Min Sketch 是一种可以处理等值查询， Join 大小估计等的数据结构，并且可以提供很强的准确性保证。Count-Min Sketch 维护了一个 d*w 的计数数组，对于每一个值，用 d 个独立的 hash 函数映射到每一行的一列中，并对应修改这 d 个位置的计数值。

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20210308153929.png)

如果对 Count-Min Sketch 更深理解感兴趣的同学，可以参考文献：[http://dimacs.rutgers.edu/~graham/pubs/papers/cm-full.pdf](http://dimacs.rutgers.edu/~graham/pubs/papers/cm-full.pdf)

### 统计信息的使用

### 范围查询

如果有一个 SQL 是类似于这种形式, select count(*) from t where a>1.7 and a<2.8; 对于某一列上的范围查询，TiDB 选择常用的等深直方图进行估算。

在前面介绍直方图时，举了一个直方图例子是包含了四个桶 [1.6, 1.9]，[2.0, 2.6]，[2.7, 2.8]，[2.9, 3.5]，桶深都是 3 的直方图。假设目前统计信息直方图就是如例子所示，查询条件是 a>1.7 and a<2.8。需要估算落在区间 [1.7,2.8] 范围内的有多少值。把这个区间对应到直方图上，可以看到 [2.0,2.6] 和 [2.7,2.8] 两个桶是完全覆盖在区间内的，估算可以直接加上两个桶的高度。但是对于区间 [1.6,1.9] 并没有完全覆盖，这时候估算 [1.7,1.9] 范围内有多少值的话就需要假设这个范围的值是连续且均匀的，那么就可以按照区间覆盖比例估算有多少值。(1.9-1.7)/(1.9-1.6) * 3 = 2。所以根据直方图估算查询条件 a>1.7 and a<2.8 这个条件共有 3+3+2 = 8 个值。

一些补充的点：

1. 对于字符串类型那个字段在计算比例时，会将字符串映射成数字，然后计算比例
2. 等深直方图每个桶的高度近似相等，直方图的边界不重合，每个值只会出现在一个桶内

### 等值查询

如果是 select count(*) from t where a = 1 这种等值查询，如果使用直方图进行估算时就会假设每个值出现的次数都是相等的，这样就可以用（总行数/不同值的数量）来估计，但是实际情况一个列上每个值很有可能出现次数是不相等的，那通过直方图这种方式估算代价，误差就会比较大。

正常使用 Count-Min Sketch 方式估算等值查询的时候，是取每一行 hash 函数对应位置的值中的最小值作为估算值。但是因为会有别的值的可能 hash 结果在同一个位置，所以 Count-Min Sketch 估计的结果总是不小于实际值。因此 TiDB 中参考文献 [http://webdocs.cs.ualberta.ca/~drafiei/papers/cmm.pdf](http://webdocs.cs.ualberta.ca/~drafiei/papers/cmm.pdf) 中提出的 Count-Mean-Min Sketch ，其与 Count-Min Sketch 在更新的时候是一样的，区别在与查询的时候：对于每一行 i，若 hash 函数映射到了值 j，那么用 (N - CM[i, j]) / (w-1)（N 是总共的插入的值数量）作为其他值产生的噪音，因此用 CM[i,j] - (N - CM[i, j]) / (w-1) 这一行的估计值，然后用所有行的估计值的中位数作为最后的估计值。

### 多列查询

前面讲解都是 TiDB 中对于单列的查询条件进行估算的方式，但是实际情况查询条件往往会包含多个列上的查询条件。在 TiDB 中采用了独立性假设的方式处理多列查询条件，就是认为不同列之间是相互独立的，因为只需要把不同列之间的过滤率乘起来就是多列查询条件的最终过滤率。

独立性假设的定义：

- selectivity 等于满足条件的行数除以表的总行数
- selectivity(a = 1 and b < 1 and c >1) = selectivity(a = 1) * selectivity(b < 1) * selectivity(c > 1)

如果是索引上的查询条件，那不需要独立性假设：例如对 (a,b) 上的索引，条件 (a = 1 and b < 1) 或者 (a = 1 and b =1) 可以用直方图或者 Count-Min Sketch 进行估算。实现独立性假设时最重要的一个任务就是将所有的查询条件分成尽量少的组，使得每一组的条件都可以用某一列或者某一索引上的统计信息进行估计，这样可以做尽量少的独立性假设。

## 统计信息日常操作

在这一部分会介绍一下在运维 TiDB 过程中与统计信息相关的一些日常操作，比如开发人员反馈 SQL 之前运行一直都很快，但是突然就运行的很慢了，这个时候就可以考虑检查一下统计信息的健康程度，是不是因为统计信息过期导致优化器选择了错误执行计划，影响了执行的效率。也比如线上环境运行 SQL 比较慢，在线上环境直接尝试优化 SQL 可能会对别的正在运行的业务造成影响，但是线上环境的数据导入到测试环境又比较麻烦，这个时候就可以考虑将线上环境的统计信息导出并导入到测试环境，方便快速验证 SQL 优化的效果。

### 统计信息收集

TiDB 中的统计信息收集根据形式划分的话可以分为手动收集统计信息和自动收集统计信息两种。手动收集统计信息又可以分为全量统计信息收集和增量统计信息收集。通过 SHOW STATS_HEALTHY 可以查看表的统计信息健康度，并粗略估计表上统计信息的准确度。当统计信息健康度 health 较低的时候，可以考虑手动收集统计信息。

### 手动收集统计信息

- 全量收集 TableNameList 中所有表的统计信息, WITH NUM BUCKETS 可以用于指定生成直方图的桶数量上限，当桶的数量越多，直方图的估算精度就越高，不过也会同时增大统计信息的内存使用，所以这个需要按照实际情况尽情调整。

```
ANALYZE TABLE TableNameList [WITH NUM BUCKETS];
```

- 全量收集 TableName 中所有 IndexNameList 中的索引列的统计信息，当 IndexNameList 为空的时候，表示收集所有索引列的统计信息

```
ANALYZE TABLE TableName INDEX [IndexNameList] [WITH NUM BUCKETS];
```

- 全量收集 TableName 中所有的 PartitionNameList 中分区的统计信息
- 全量收集 TableName 中所有的 PartitionNameList 中分区的索引列统计信息

对于类似时间列这样的单调不减列，在进行全量收集后，可以使用增量收集来单独分析新增的部分，以提高分析的速度。需要注意的点：

1. 目前只对索引列的统计信息提供增量收集的功能
2. 使用增量收集时，必须保证表上只有插入操作，且应用需要保证索引列上新插入的值是单调不减的，否则会导致统计信息不准，影响 TiDB 优化器选择合适的执行计划。
- 增量收集 TableName 中所有的 IndexNameList 中的索引列的统计信息

```
ANALYZE INCREMENTAL TABLE TableName INDEX [IndexNameList] [WITH NUM BUCKETS];
```

- 增量收集 TableName 中所有的 PartitionNameList 中分区的索引列统计信息

```
ANALYZE INCREMENTAL TABLE TableName PARTITION PartitionNameList INDEX [IndexNameList] [WITH NUM BUCKETS];
```

### 自动收集统计信息

自动收集统计信息相关参数：

- tidb_auto_analyze_ratio: 自动更新的阈值，当表中的修改行数量占总行数的比例超过该参数设置值，则会触发自动收集统计信息。
- tidb_auto_analyze_start_time: 自动更新开始的时间，用于控制一天中在什么时间点开始可以触发自动收集统计信息操作。默认值 00:00 +0000
- tidb_auto_analyze_end_time: 自动更新结束的时间，用于控制一天中在什么时间点结束自动收集统计信息操作。默认值 23:59 +0000
- feedback-probability: 对于每一个查询，TiDB 会以该参数设置的概率收集反馈信息，并将其用于更新直方图和 Count-Min Sketch。默认值是 0.05， 设置为 0.0 表示关闭 feedback 功能。

当某个表 tbl 的修改行数与总行数的比值大于 tidb_auto_analyze_ratio，并且当前时间在 tidb_auto_analyze_start_time 和 tidb_auto_analyze_end_time 之间时，TiDB 会在后台执行 ANALYZE TABLE tbl 语句自动更新这个表的统计信息。

### 控制 ANALYZE 并发度

在 TiDB 中执行 ANALYZE TABLE 语句会比 MySQL 中执行相同语句耗时更长。因为 MySQL InnoDB 引擎只是采样少量的 page 页用于收集统计信息，但是 TiDB 会扫描所有的数据用于更新统计信息。

相关参数：

- tidb_enable_fast_analyze: 控制 TiDB 是否开启快速分析功能，设置为 1 开启该功能后，TiDB 会随机采样约 10000 行数据来构建统计信息，但是相对统计信息的准确度会比较差。默认值是 0，表示不开启快速分析功能。
- tidb_build_stats_concurrency: 控制收集统计信息的并发度，默认值是 4。
- tidb_distsql_scan_concurrency: 在执行分析普通列任务的时候，该参数用于控制一次读取的 Region 数量，默认值为 15. 该参数不建议设置过大，不超过所有 TiKV CPU 核数总和。
- tidb_index_serial_scan_concurrency: 在执行分析索引列任务的时候，该参数用于控制一次读取的 Region 数量，默认值是 1。

### 查看 ANALYZE 状态

```
SHOW ANALYZE STATUS [ShowLikeOrWhere];
```

输出结果

![](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20210308154205.png)

### 统计信息导入导出

导出统计信息

- 导出 ${db_name} 中表 ${table_name} 的 json 格式统计信息

```
http://${tidb-server-ip}:${tidb-server-status-port}/stats/dump/${db_name}/${table_name}
```

- 导出 ${db_name} 中表 ${table_name} 的指定时间(两种形式)的 json 格式统计信息，需要注意指定时间需要在 GC SafePoint 之后

```
http://${tidb-server-ip}:${tidb-server-status-port}/stats/dump/${db_name}/${table_name}/${yyyyMMddHHmmss}
http://${tidb-server-ip}:${tidb-server-status-port}/stats/dump/${db_name}/${table_name}/${yyyy-MM-dd HH:mm:ss}
```

导入统计信息

- 导入统计信息，file_name 是要导入的统计信息的文件名

```
LOAD STATS 'file_name';
```