概述 : 

解决一个问题,首先得知道这个是啥问题,没有具体场景下的hive如何调优,首先要解决问题的本质:调优的方向是啥?

在这个问题之前,首先大体知道Hive是啥 : hive是hadoop生态,java语言编写,基于hdfs文件系统,yarn调度系统,有多种文件格式和压缩类型,可以对接多种计算引擎(MR,Spark,Impla等)的大数据存储,将sql结合配置和优化器,主要解析成为MR任务,并在读时校验格式的离线报表计算的行业标准. 

在这个大体概念的基础上,可以推出hive的方向是大数据量尽快更节约资源的计算,hive调优的方向也就是这两点,在兼顾资源的基础上,尽量降低整体计算时间.类似围着操场跑一圈,有下面几个方案:

| 思路         | 具体方案                                                     |
| ------------ | ------------------------------------------------------------ |
| 能不跑就不跑 | bit map索引/orc bloom索引/orc文件的元数据有某字段在此文件的区间,与clickhouse类似 |
| 能少跑就少跑 | 表层面的分区/分桶优化/谓词下推提前过滤/列,行,分区裁剪,分桶定位 |
| 并发跑       | map,reduce数配置/不同不相互依赖的stage开启并发处理/向量化执行,一次处理一个批次 |
| 中间过程更快 | 中间结果压缩(shuffle过程网络IO),压缩和CPU的权衡/join左侧维度表自动开启是否放入内存join的判断 |
| 运行资源节省 | 多个group by的连接键统一>一个map任务/jvm重用/本地化执行小sql/一个from跟2个insert/yarn资源队列优化 |
| 存储资源节省 | 文件的格式和压缩格式/冷数据处理方案/合并小文件(namenode压力/map端输入/map,reduce输出) |

# 1、表层面

## 1.1 利用分区表优化

分区表 是在某一个或者几个维度上对数据进行分类存储，一个分区对应一个目录。如果筛选条件里有分区字段，那么 Hive 只需要遍历对应分区目录下的文件即可，不需要遍历全局数据，使得处理的数据量大大减少，从而提高查询效率。

也就是说：当一个 Hive 表的查询大多数情况下，会根据某一个字段进行筛选时，那么非常适合创建为分区表，该字段即为分区字段。

eg:

```
CREATE TABLE page_view
(viewTime INT, 
 userid BIGINT,
 page_url STRING, 
 referrer_url STRING,
 ip STRING COMMENT 'IP Address of the User')
PARTITIONED BY(date STRING, country STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '1'
STORED AS TEXTFILE;
```

**1、当你意识到一个字段经常用来做where，建分区表，使用这个字段当做分区字段
2、在查询的时候，使用分区字段来过滤，就可以避免全表扫描。只需要扫描这张表的一个分区的数据即可**

## 1.2 利用分桶表优化

**跟分区的概念很相似，都是把数据分成多个不同的类别**

1、分区：按照字段值来进行，一个分区，就只是包含这个值的所有记录

不是当前分区的数据一定不在当前分区

当前分区也只会包含当前这个分区值的数据

2、分桶：默认规则，Hash的方式

一个桶中会有多个不同的值

如果一个分桶中，包含了某个值，这个值的所有记录，必然都在这个分桶里面

Hive Bucket，分桶，是指将数据以指定列的值为key进行hash，hash到指定数目的桶里面，这样做的目的和分区表类似，是的筛选时不用全局遍历所有的数据，只需要遍历所在的桶就好了，这样也只可以支持高效采样。

其实最主要的作用就是 **采样、join**

**如下例就是以 userid 这一列为 bucket 的依据，共设置 32 个 buckets**

```
CREATE TABLE page_view(viewTime INT, 
userid BIGINT,
page_url STRING, 
referrer_url STRING,
ip STRING COMMENT 'IP Address of the User'
)COMMENT 'This is the page view table' 
PARTITIONED BY(dt STRING, country STRING) 
CLUSTERED BY(userid) 
SORTED BY(viewTime) 
INTO 32 BUCKETS;
         
#创建分桶表
CREATE TABLE table_name 
PARTITIONED BY (partition1 data_type, partition2 data_type,….) CLUSTERED BY (column_name1, column_name2, …) 
SORTED BY (column_name [ASC|DESC], …)] 
INTO num_buckets BUCKETS;
```

**CLUSTERED BY(userid)：** 按照userid进行分桶

**SORTED BY(viewTime)：** 按照viewTime进行桶内排序

**INTO 32 BUCKETS：** 分成多少个桶

**两个表以相同方式（相同字段）划分桶，两个表的桶个数一定是倍数关系，这样在join的时候速度会大大增加**

采样用的不多，也就不过多阐述了

## 1.3 选择合适的文件存储格式

在 HiveSQL 的 create table 语句中，可以使用 stored as … 指定表的存储格式。Apache Hive支持 Apache Hadoop 中使用的几种熟悉的文件格式，比如 TextFile、SequenceFile、RCFile、Avro、ORC、ParquetFile等。存储格式一般需要根据业务进行选择，在我们的实操中，绝大多数表都采用TextFile与Parquet两种存储格式之一。TextFile是最简单的存储格式，它是纯文本记录，也是Hive的默认格式。虽然它的磁盘开销比较大，查询效率也低，但它更多地是作为跳板来使用。RCFile、ORC、Parquet等格式的表都不能由文件直接导入数据，**必须由TextFile来做中转。Parquet和ORC都是Apache旗下的开源列式存储格式。列式存储比起传统的行式存储更适合批量OLAP查询，并且也支持更好的压缩和编码。创建表时，特别是宽表，尽量使用 ORC、ParquetFile 这些列式存储格式**，因为列式存储的表，每一列的数据在物理上是存储在一起的，Hive查询时会只遍历需要列数据，大大减少处理的数据量。

如果数据存储在小于块大小的小文件中，则可以使用**SEQUENCE**文件格式。如果要以减少存储空间并提高性能的优化方式存储数据，则可以使用**ORC文件**格式，而当列中嵌套的数据过多时，**Parquet**格式会很有用。因此，需要根据拥有的数据确定输入文件格式。

![hive orc](Optimize.assets/20160401-11.jpg)

**TextFile**

1、存储方式：行存储。默认格式，如果建表时不指定默认为此格式。，
2、每一行都是一条记录，每行都以换行符"\n"结尾。数据不做压缩时，磁盘会开销比较大，数据解析开销也
比较大。
3、可结合Gzip、Bzip2等压缩方式一起使用（系统会自动检查，查询时会自动解压）,推荐选用可切分的压
缩算法。

**Sequence File**

1、一种Hadoop API提供的二进制文件，使用方便、可分割、可压缩的特点。
2、支持三种压缩选择：NONE、RECORD、BLOCK。RECORD压缩率低，一般建议使用BLOCK压缩

**RC File**

1、存储方式：数据按行分块，每块按照列存储 。
A、首先，将数据按行分块，保证同一个record在一个块上，避免读一个记录需要读取多个block。
B、其次，块数据列式存储，有利于数据压缩和快速的列存取。
2、相对来说，RCFile对于提升任务执行性能提升不大，但是能节省一些存储空间。可以使用升级版的ORC格
式。

**ORC File**

1、存储方式：数据按行分块，每块按照列存储
2、Hive提供的新格式，属于RCFile的升级版，性能有大幅度提升，而且数据可以压缩存储，压缩快，快速列存取。
3、ORC File会基于列创建索引，当查询的时候会很快。

**Parquet File**

1、存储方式：列式存储。
2、Parquet对于大型查询的类型是高效的。对于扫描特定表格中的特定列查询，Parquet特别有用。Parquet一般使用Snappy、Gzip压缩。默认Snappy。
3、Parquet支持Impala 查询引擎。
4、表的文件存储格式尽量采用Parquet或ORC，不仅降低存储量，还优化了查询，压缩，表关联等性能

## 1.4 选择合适的压缩格式

Hive 语句最终是转化为 MapReduce 程序来执行的，而 MapReduce 的性能瓶颈在与 网络IO 和 磁盘IO，要解决性能瓶颈，最主要的是 减少数据量，对数据进行压缩是个好方式。压缩虽然是减少了数据量，但是压缩过程要消耗 CPU，但是在 Hadoop 中，往往性能瓶颈不在于 CPU，CPU 压力并不大，所以压缩充分利用了比较空闲的 CPU。

**常用压缩方法对比**

org.apache.hadoop.io.compress.DefaultCodec
org.apache.hadoop.io.compress.GzipCodec
org.apache.hadoop.io.compress.BZip2Codec
com.hadoop.compression.lzo.LzopCodec
org.apache.hadoop.io.compress.Lz4Codec
org.apache.hadoop.io.compress.SnappyCodec

| 压缩格式 | 是否可拆分 | 是否自带 | 压缩率 | 速度   | 是否hadoop自带 |
| -------- | ---------- | -------- | ------ | ------ | -------------- |
| gzip     | 否         | 是       | 很高   | 比较快 | 是             |
| lzo      | 是         | 是       | 比较高 | 很快   | 否             |
| snappy   | 否         | 是       | 比较高 | 很快   | 否             |
| bzip2    | 是         | 否       | 最高   | 慢     | 是             |

**压缩率对比**

![图片](Optimize.assets/640.webp)

> 如何选择压缩方式呢？

1、压缩比例

2、解压缩速度

3、是否支持split

**支持切割的文件可以并行的有多个mapper程序处理大数据文件，一般我们选择的都是支持切分的！**

> 压缩带来的缺点和优点

1、计算密集型，不压缩，否则会进一步增加cpu的负担，真实的场景中hive对cpu的压力很小

2、网络密集型，推荐压缩，减小网络数据传输

```
# Job 输出文件按照 Block## 默认值是false
set mapreduce.output.fileoutputformat.compress=true;   
## 默认值是Record
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
set mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.lzo.LzoCodec;
# Map 输出结结果进行压缩
set mapred.map.output.compress=true;
set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.lzo.LzoCodec;
# 对 Hive 输出结果和中间都进行压缩
set hive.exec.compress.output=true  ## 默认值是false，不压缩
set hive.exec.compress.intermediate=true   ## 默认值是false，为true时MR设置的压缩才启用
```

# 2、HQL层面优化

## 2.1 执行计划

```
explain select * from movies;
```

![图片](Optimize.assets/641.webp)

## 2.1 列、行、分区裁剪

列裁剪就是在查询时只读取需要的列

行裁剪就是在查询时只读取需要的行，也就是提前过滤

分区剪裁就是在查询的时候只读取需要的分区。

```
set hive.optimize.cp = true; 列裁剪，取数只取查询中需要用到的列，默认是true
set hive.optimize.pruner=true;   ## 分区剪裁
```

## 2.2 谓词下推

将 SQL 语句中的 where 谓词逻辑都尽可能提前执行，减少下游处理的数据量。对应逻辑优化器是PredicatePushDown。

```
set hive.optimize.ppd=true;   ## 默认是true
```

**eg:**

```
select a.*, b.* from a join b on a.id = b.id where b.age > 20;
```

转换为下面的这样的

```
select a.*, c.* from a join (select * from b where age > 20) c on a.id = c.id;
```

## 2.3 合并小文件

如果一个mapreduce job碰到一对小文件作为输入，一个小文件启动一个Task，这样的话会出现maptask爆炸的问题。

**Map端输入合并**

在执行 MapReduce 程序的时候，一般情况是一个文件的一个数据分块需要一个 mapTask 来处理。但是如果数据源是大量的小文件，这样就会启动大量的 mapTask 任务，这样会浪费大量资源。可以将输入的小文件进行合并，从而减少 mapTask 任务数量。

```
## Map端输入、合并文件之后按照block的大小分割（默认）set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;## Map端输入，不合并set hive.input.format=org.apache.hadoop.hive.ql.io.HiveInputFormat;
```

**Map/Reduce输出合并**

大量的小文件会给 HDFS 带来压力，影响处理效率。可以通过合并 Map 和 Reduce 的结果文件来消除影响

```
## 是否合并Map输出文件, 默认值为true
set hive.merge.mapfiles=true;
## 是否合并Reduce端输出文件,默认值为false
set hive.merge.mapredfiles=true;
## 合并文件的大小,默认值为256000000 256M
set hive.merge.size.per.task=256000000;
## 每个Map 最大分割大小
set mapred.max.split.size=256000000; 
## 一个节点上split的最少值
set mapred.min.split.size.per.node=1;  // 服务器节点## 一个机架上split的最少值set mapred.min.split.size.per.rack=1;   // 服务器机架
```

`hive.merge.size.per.task` 和 `mapred.min.split.size.per.node` 联合起来：

**1、默认情况先把这个节点上的所有数据进行合并，如果合并的那个文件的大小超过了256M就开启另外一个文件继续合并**
**2、如果当前这个节点上的数据不足256M，那么就都合并成一个逻辑切片。**

## 2.4 合理设置MapTask并行度

**Map数过大** ：当输入文件特别大，MapTask 特别多，每个计算节点分配执行的 MapTask 都很多，这时候可以考虑减少 MapTask 的数量。增大每个 MapTask 处理的数据量。而且 MapTask 过多，最终生成的结果文件数也太多。

**1、Map阶段输出文件太小，产生大量小文件**
**2、初始化和创建Map的开销很大**

**Map数太小** ：当输入文件都很大，任务逻辑复杂，MapTask 执行非常慢的时候，可以考虑增加MapTask 数，来使得每个 MapTask 处理的数据量减少，从而提高任务的执行效率。

**1、文件处理或查询并发度小，Job执行时间过长**
**2、大量作业时，容易堵塞集群**

**一个MapReduce Job 的 MapTask 数量是由输入分片InputSplit 决定的。而输入分片是由 FileInputFormat.getSplit() 决定的。一个输入分片对应一个MapTask，而输入分片是由三个参数决定的：**

| 参数                                          | 默认值 | 意义               |
| --------------------------------------------- | ------ | ------------------ |
| dfs.blocksize                                 | 128M   | HDFS默认数据块大小 |
| mapreduce.input.fileinputformat.split.minsize | 1      | 最小分片大小(MR)   |
| mapreduce.input.fileinputformat.split.maxsize | 256M   | 最大分片大小(MR)   |

输入分片大小的计算是这么计算出来的：

```
long splitSize = Math.max(minSize, Math.min(maxSize, blockSize))
```

默认情况下，输入分片大小和 HDFS 集群默认数据块大小一致，也就是默认一个数据块，启用一个MapTask 进行处理，这样做的好处是避免了服务器节点之间的数据传输，提高 job 处理效率

**两种经典的控制MapTask的个数方案：减少MapTask数 或者 增加MapTask数**

1、减少 MapTask 数是通过合并小文件来实现，这一点主要是针对数据源
2、增加 MapTask 数可以通过控制上一个 job 的 reduceTask 个数
**重点注意：不推荐把这个值进行随意设置！**
**推荐的方式：使用默认的切块大小即可。如果非要调整，最好是切块的N倍数**

> 最好的方式就是 NodeManager节点个数：N ===》 Task = ( N * 0.95) * MapTask

**合理控制 MapTask 数量**

1、减少 MapTask 数可以通过合并小文件来实现
2、增加 MapTask 数可以通过控制上一个 ReduceTask 默认的 MapTask 个数

```
输入文件总大小：total_size  
HDFS 设置的数据块大小：dfs_block_size   
default_mapper_num = total_size / dfs_block_size
```

MapReduce 中提供了如下参数来控制 map 任务个数，从字面上看，貌似是可以直接设置 MapTask 个数的样子，但是很遗憾不行，这个参数设置只有在大于 default_mapper_num 的时候，才会生效

```
set mapred.map.tasks=10; ## 默认值是2
```

那如果我们需要减少 MapTask 数量，但是文件大小是固定的，那该怎么办呢?可以通过 mapred.min.split.size 设置每个任务处理的文件的大小，这个大小只有在大于dfs_block_size 的时候才会生效

```
split_size = max(mapred.min.split.size, dfs_block_size)split_num = total_size / split_size
compute_map_num = Math.min(split_num, Math.max(default_mapper_num,mapred.map.tasks))
```

这样就可以减少 MapTask 数量了

让我们来总结一下控制mapper个数的方法：

```
1、如果想增加 MapTask 个数，可以设置 mapred.map.tasks 为一个较大的值2、如果想减少 MapTask 个数，可以设置 maperd.min.split.size 为一个较大的值3、如果输入是大量小文件，想减少 mapper 个数，可以通过设置 hive.input.format 合并小文
```

如果想要调整 mapper 个数，在调整之前，需要确定处理的文件大概大小以及文件的存在形式（是大量小文件，还是单个大文件），然后再设置合适的参数。不能盲目进行暴力设置，不然适得其反。

MapTask 数量与输入文件的 split 数息息相关，在 Hadoop 源码`org.apache.hadoop.mapreduce.lib.input.FileInputFormat` 类中可以看到 split 划分的具体逻辑。可以直接通过参数 `mapred.map.tasks`（默认值2）来设定 MapTask 数的期望值，但它不一定会生效。

## 2.5 合理设置ReduceTask并行度

如果 ReduceTask 数量过多，一个 ReduceTask 会产生一个结果文件，这样就会生成很多小文件，那么如果这些结果文件会作为下一个 Job 的输入，则会出现小文件需要进行合并的问题，而且启动和初始化ReduceTask 需要耗费资源。

如果 ReduceTask 数量过少，这样一个 ReduceTask 就需要处理大量的数据，并且还有可能会出现数据倾斜的问题，使得整个查询耗时长。默认情况下，Hive 分配的 reducer 个数由下列参数决定：

Hadoop MapReduce 程序中，ReducerTask 个数的设定极大影响执行效率，ReducerTask 数量与输出文件的数量相关。如果 ReducerTask 数太多，会产生大量小文件，对HDFS造成压力。如果ReducerTask 数太少，每个ReducerTask 要处理很多数据，容易拖慢运行时间或者造成 OOM。这使得Hive 怎样决定 ReducerTask 个数成为一个关键问题。遗憾的是 Hive 的估计机制很弱，不指定ReducerTask 个数的情况下，Hive 会猜测确定一个ReducerTask 个数，基于以下两个设定：

```
参数1：hive.exec.reducers.bytes.per.reducer (默认256M)
参数2：hive.exec.reducers.max (默认为1009)
参数3：mapreduce.job.reduces (默认值为-1，表示没有设置，那么就按照以上两个参数进行设置)
```

ReduceTask 的计算公式为:

```
N = Math.min(参数2，总输入数据大小 / 参数1)
```

可以通过改变上述两个参数的值来控制 ReduceTask 的数量。也可以通过

```
set mapred.map.tasks=10;
set mapreduce.job.reduces=10;
```

通常情况下，有必要手动指定 ReduceTask 个数。考虑到 Mapper 阶段的输出数据量通常会比输入有大幅减少，因此即使不设定 ReduceTask 个数，重设 参数2 还是必要的。依据经验，可以将 参数2 设定为 M * （0.95 * N） (N为集群中 NodeManager 个数)。一般来说，NodeManage 和 DataNode 的个数是一样的

## 2.6 Join优化

**1. Join的整体优化原则：**

1、优先过滤后再进行Join操作，最大限度的减少参与join的数据量
2、小表join大表，最好启动mapjoin，hive自动启用mapjoin, 小表不能超过25M，可以更改
3、Join on的条件相同的话，最好放入同一个job，并且join表的排列顺序从小到大：

```
select a.*,b.*, c.* from a join b on a.id = b.id join c on a.id = c.i
```

4、如果多张表做join, 如果多个链接条件都相同，会转换成一个JOb

**2. 优先过滤数据：**

尽量减少每个阶段的数据量，对于分区表能用上分区字段的尽量使用，同时只选择后面需要使用到的列，最大限度的减少参与 Join 的数据量

**3. 小表join大表的原则：**

小表 join 大表的时应遵守小表 join 大表原则，原因是 join 操作的 reduce 阶段，位于 join 左边的表内容会被加载进内存，将条目少的表放在左边，可以有效减少发生内存溢出的几率。join 中执行顺序是从左到右生成 Job，应该保证连续查询中的表的大小从左到右是依次增加的。

**4. 使用相同的连接键：**

在 hive 中，当对 3 个或更多张表进行 join 时，如果 on 条件使用相同字段，那么它们会合并为一个MapReduce Job，利用这种特性，可以将相同的 join on 放入一个 job 来节省执行时间。

**5. 尽量原子操作：**

尽量避免一个SQL包含复杂的逻辑，可以使用中间表来完成复杂的逻辑。

**6. 大表join大表：**

1、空key过滤：有时join超时是因为某些key对应的数据太多，而相同key对应的数据都会发送到相同的reducer上，从而导致内存不够。此时我们应该仔细分析这些异常的key，很多情况下，这些key对应的数据是异常数据，我们需要在SQL语句中进行过滤。

2、空key转换：有时虽然某个key为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在join的结果中，此时我们可以表a中key为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的reducer上

**7. 启用MapJoin：**

这个优化措施，只要能用的时候一定要用，根据数据量大小来调整小表的大小，一般公司里面可以设置到512 到1G

MapJoin 是将 join 双方比较小的表直接分发到各个 map 进程的内存中，在 map 进程中进行 join 操作，这样就不用进行 reduce 步骤，从而提高了速度。只有 join 操作才能启用 MapJoin。

```
## 是否根据输入小表的大小，自动将reduce端的common join 转化为map join，将小表刷入内存中。## 对应逻辑优化器是MapJoinProcessorset hive.auto.convert.join = true;## 刷入内存表的大小(字节)set hive.mapjoin.smalltable.filesize = 25000000;## hive会基于表的size自动的将普通join转换成mapjoinset hive.auto.convert.join.noconditionaltask=true;## 多大的表可以自动触发放到内层LocalTask中，默认大小10Mset hive.auto.convert.join.noconditionaltask.size=10000000;
```

Hive 可以进行多表 Join。Join 操作尤其是 Join 大表的时候代价是非常大的。MapJoin 特别适合大小表join的情况。在Hive join场景中，一般总有一张相对小的表和一张相对大的表，小表叫 build table，大表叫 probe table。Hive 在解析带 join 的 SQL 语句时，会默认将最后一个表作为 probe table，将前面的表作为 build table 并试图将它们读进内存。如果表顺序写反，probe table 在前面，引发 OOM 的风险就高了。在维度建模数据仓库中，事实表就是 probe table，维度表就是 build table。这种 Join 方式在 map 端直接完成 join 过程，消灭了 reduce，效率很高。而且 MapJoin 还支持非等值连接。当 Hive 执行 Join 时，需要选择哪个表被流式传输（stream），哪个表被缓存（cache）。Hive 将JOIN 语句中的最后一个表用于流式传输，因此我们需要确保这个流表在两者之间是最大的。如果要在
不同的 key 上 join 更多的表，那么对于每个 join 集，只需在 ON 条件右侧指定较大的表

也可以手动开启mapjoin：

```
--SQL方式，在SQL语句中添加MapJoin标记（mapjoin hint）--将小表放到内存中，省去shffle操作// 在没有开启mapjoin的情况下，执行的是reduceJoinSELECT  /*+ MAPJOIN(smallTable) */ smallTable.key, bigTable.value FROMsmallTable  JOIN bigTable  ON smallTable.key = bigTable.key;
```

**在高版本中，已经进行了优化，会自动进行优化**

**8. Sort-Merge-Bucket(SMB) Map Join：**

它是另一种Hive Join优化技术，使用这个技术的前提是所有的表都必须是分桶表（bucket）和分桶排序的（sort）。分桶表的优化！

具体实现：

```
1、针对参与join的这两张做相同的hash散列，每个桶里面的数据还要排序2、这两张表的分桶个数要成倍数。3、开启 SMB join 的开关！
```

一些常见的参数设置：

```
## 当用户执行bucket map join的时候，发现不能执行时，禁止查询set hive.enforce.sortmergebucketmapjoin=false; ## 如果join的表通过sort merge join的条件，join是否会自动转换为sort merge joinset hive.auto.convert.sortmerge.join=true;## 当两个分桶表 join 时，如果 join on的是分桶字段，小表的分桶数是大表的倍数时，可以启用mapjoin 来提高效率。# bucket map join优化，默认值是 falseset hive.optimize.bucketmapjoin=false; ## bucket map join 优化，默认值是 falseset hive.optimize.bucketmapjoin.sortedmerge=false;
```

**9. Join数据倾斜优化:**

**在编写 Join 查询语句时，如果确定是由于 join 出现的数据倾斜，那么请做如下设置：**

```
# join的键对应的记录条数超过这个值则会进行分拆，值根据具体数据量设置set hive.skewjoin.key=100000;  # 如果是join过程出现倾斜应该设置为trueset hive.optimize.skewjoin=false;
```

如果开启了，在 Join 过程中 Hive 会将计数超过阈值 hive.skewjoin.key（默认100000）的倾斜 key 对应的行临时写进文件中，然后再启动另一个 job 做 map join 生成结果。通过 `hive.skewjoin.mapjoin.map.tasks` 参数还可以控制第二个 job 的 mapper 数量，默认10000。

例如`set hive.skewjoin.mapjoin.map.tasks=10000;`

## 2.7 CBO优化

join的时候表的顺序的关系：前面的表都会被加载到内存中。后面的表进行磁盘扫描

```
select a.*, b.*, c.* from a join b on a.id = b.id join c on a.id = c.id;
```

Hive 自 0.14.0 开始，加入了一项 “Cost based Optimizer” 来对 HQL 执行计划进行优化，这个功能通过 “hive.cbo.enable” 来开启。在 Hive 1.1.0 之后，这个 feature 是默认开启的，它可以 **自动优化 HQL中多个 Join 的顺序，并选择合适的 Join 算法。**

CBO，成本优化器，代价最小的执行计划就是最好的执行计划。传统的数据库，成本优化器做出最优化的执行计划是依据统计信息来计算的。Hive 的成本优化器也一样。Hive 在提供最终执行前，优化每个查询的执行逻辑和物理执行计划。这些优化工作是交给底层来完成的。根据查询成本执行进一步的优化，从而产生潜在的不同决策：如何排序连接，执行哪种类型的连接，并行度等等。要使用基于成本的优化（也称为CBO），请在查询开始设置以下参数：

```
set hive.cbo.enable=true;set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
```

## 2.8 Group By优化

默认情况下，Map 阶段同一个 Key 的数据会分发到一个 Reduce 上，当一个 Key 的数据过大时会产生数据倾斜。进行 group by 操作时可以从以下两个方面进行优化：

1、Map端部分预聚合：

事实上并不是所有的聚合操作都需要在 Reduce 部分进行，很多聚合操作都可以先在 Map 端进行部分聚合，然后在 Reduce 端的得出最终结果。

```
## 开启Map端聚合参数设置set hive.map.aggr=true;
# 设置map端预聚合的行数阈值，超过该值就会分拆job，默认值100000
set hive.groupby.mapaggr.checkinterval=100000  
```

2、有数据倾斜时进行负载均衡

当 HQL 语句使用 group by 时数据出现倾斜时，如果该变量设置为 true，那么 Hive 会自动进行负载均衡。策略就是把 MapReduce 任务拆分成两个：第一个先做预汇总，第二个再做最终汇总。

```
# 自动优化，有数据倾斜的时候进行负载均衡（默认是false） 如果开启设置为trueset hive.groupby.skewindata=false;
```

当选项设定为 true 时，生成的查询计划有两个 MapReduce 任务。

```
1、在第一个 MapReduce 任务中，map 的输出结果会随机分布到 reduce 中，每个 reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的`group by key`有可能分发到不同的 reduce 中，从而达到负载均衡的目的；2、第二个 MapReduce 任务再根据预处理的数据结果按照 group by key 分布到各个 reduce 中，最后完成最终的聚合操作。
```

Map 端部分聚合：并不是所有的聚合操作都需要在 Reduce 端完成，很多聚合操作都可以先在 Map 端进行部分聚合，最后在 Reduce 端得出最终结果，对应的优化器为 GroupByOptimizer。

**那么如何用 group by 方式同时统计多个列？**

简单版：

```
select t.a, sum(t.b), count(t.c), count(t.d) from some_table t group by t.a;
```

优化版：

```
select t.a, sum(t.b), count(t.c), count(t.d) from (
  select a,b,null c,null d from some_table  union all
  select a,0 b,c,null d from some_table group by a,c  union all
  select a,0 b,null c,d from some_table group by a,d) t;
```

## 2.9 Order By优化

order by 只能是在一个 reduce 进程中进行，所以如果对一个大数据集进行 order by ，会导致一个reduce 进程中处理的数据相当大，造成查询执行缓慢。

```
1、在最终结果上进行order by，不要在中间的大数据集上进行排序。如果最终结果较少，可以在一个reduce上进行排序时，那么就在最后的结果集上进行order by。
2、如果是取排序后的前N条数据，可以使用distribute by和sort by在各个reduce上进行排序后前N条，然后再对各个reduce的结果集合合并后在一个reduce中全局排序，再取前N条，因为参与全局排序的order by的数据量最多是reduce个数 * N，所以执行效率会有很大提升。
```

**在Hive中，关于数据排序，提供了四种语法，一定要区分这四种排序的使用方式和适用场景**

```
1、order by：全局排序，缺陷是只能使用一个reduce
2、sort by：单机排序，单个reduce结果有序
3、cluster by：对同一字段分桶并排序，不能和sort by连用
4、distribute by + sort by：分桶，保证同一字段值只存在一个结果文件当中，结合sort by保证每个reduceTask结果有序

创建分桶表
CREATE TABLE table_name 
PARTITIONED BY (partition1 data_type, partition2 data_type,….) CLUSTERED BY (column_name1, column_name2, …) 
SORTED BY (column_name [ASC|DESC], …)] 
INTO num_buckets BUCKETS;
```

Hive HQL 中的 order by 与其他 SQL 方言中的功能一样，就是将结果按某字段全局排序，这会导致所有 map 端数据都进入一个 reducer 中，在数据量大时可能会长时间计算不完。
如果使用 sort by，那么还是会视情况启动多个 reducer 进行排序，并且保证每个 reducer 内局部有序。为了控制map 端数据分配到 reducer 的 key，往往还要配合 distribute by 一同使用。如果不加distribute by 的话，map 端数据就会随机分配到 reducer.

1、方式一：

```
-- 直接使用order by来做。如果结果数据量很大，这个任务的执行效率会非常低
select id,name,age from student order by age desc limit 3;
```

**2、方式二：**

```
set mapreduce.job.reduces=3;
select * from student distribute by (case when age > 20 then 0 when age < 18 then 2 else 1 end) sort by (age desc);
```

**关于分界值的确定，使用采样的方式，来估计数据分布规律。**

## 2.10 Count Distinct 优化

当要统计某一列去重数时，如果数据量很大，count(distinct) 就会非常慢，原因与 order by 类似，count(distinct) 逻辑只会有很少的 reducer 来处理。这时可以用 group by 来改写：

```
-- 先 group by 在 count 
select count(1) from (
  select age from student  where department >= "MA"
  group by age) t;
```

## 2.11 怎样写in/exists语句

在Hive的早期版本中，in/exists语法是不被支持的，但是从 hive-0.8x 以后就开始支持这个语法。但是不推荐使用这个语法。虽然经过测验，Hive-2.3.6 也支持 in/exists 操作，但还是推荐使用 Hive 的一个高效替代方案：**left semi join**

```
-- in / exists 实现
select a.id, a.name from a where a.id in (select b.id from b);
select a.id, a.name from a where exists (select id from b where a.id = b.id);
```

应该转换成：

```
-- left semi join 实现
select a.id, a.name from a left semi join b on a.id = b.id;
```

**需要注意的是，一定要展示的数据只有左表中的数据！**

## 2.12 使用 vectorization 矢量查询技术

在计算类似 scan, filter, aggregation 的时候， vectorization 技术以设置批处理的增量大小为 1024 行单次来达到比单条记录单次获得更高的效率。

```
set hive.vectorized.execution.enabled=true ;set hive.vectorized.execution.reduce.enabled=true;
```

具体参考https://cwiki.apache.org/confluence/display/Hive/Vectorized+Query+Execution

## 2.13 多重插入模式

如果你碰到一堆SQL，并且这一堆SQL的模式还一样。都是从同一个表进行扫描，做不同的逻辑。可优化的地方：如果有n条SQL，每个SQL执行都会扫描一次这张表

如果一个 HQL 底层要执行 10 个 Job，那么能优化成 8 个一般来说，肯定能有所提高，多重插入就是一个非常实用的技能。一次读取，多次插入，有些场景是从一张表读取数据后，要多次利用，这时可以使用 multi insert 语法：

```
from sale_detail insert overwrite table sale_detail_multi partition (sale_date='2019',region='china' )
 select shop_name, customer_id, total_price where .....
 insert overwrite table sale_detail_multi partition (sale_date='2020',region='china' )
 select shop_name, customer_id, total_price where .....;
```

**说明：multi insert语法有一些限制。**

```
1、一般情况下，单个SQL中最多可以写128路输出，超过128路，则报语法错误。2、在一个multi insert中：
对于分区表，同一个目标分区不允许出现多次。
对于未分区表，该表不能出现多次。3、对于同一张分区表的不同分区，不能同时有insert overwrite和insert into操作，否则报错返回
```

Multi-Group by 是 Hive 的一个非常好的特性，它使得 Hive 中利用中间结果变得非常方便。例如：

```
FROM (SELECT a.status, b.school, b.gender FROM status_updates a JOIN profiles bON (a.userid = b.userid and a.ds='2019-03-20' )) subq1
INSERT OVERWRITE TABLE gender_summary PARTITION(ds='2019-03-20')SELECT subq1.gender, COUNT(1) GROUP BY subq1.gender
INSERT OVERWRITE TABLE school_summary PARTITION(ds='2019-03-20')SELECT subq1.school, COUNT(1) GROUP BY subq1.school;
```

上述查询语句使用了 Multi-Group by 特性连续 group by 了 2 次数据，使用不同的 Multi-Group by。这一特性可以减少一次 MapReduce 操作。

## 2.14 启动中间结果压缩

**map 输出压缩**

```
set mapreduce.map.output.compress=true;
set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;
```

**中间数据压缩**

中间数据压缩就是对 hive 查询的多个 Job 之间的数据进行压缩。最好是选择一个节省CPU耗时的压缩方式。可以采用 snappy 压缩算法，该算法的压缩和解压效率都非常高。

```
set hive.exec.compress.intermediate=true;
set hive.intermediate.compression.codec=org.apache.hadoop.io.compress.SnappyCodec;
set hive.intermediate.compression.type=BLOCK;
```

**结果数据压缩**

最终的结果数据（Reducer输出数据）也是可以进行压缩的，可以选择一个压缩效果比较好的，可以减少数据的大小和数据的磁盘读写时间；注：常用的 gzip，snappy 压缩算法是不支持并行处理的，如果数据源是 gzip/snappy压缩文件大文件，这样只会有有个 mapper 来处理这个文件，会严重影响查询效率。所以如果结果数据需要作为其他查询任务的数据源，可以选择支持 splitable 的 LZO 算法，这样既能对结果文件进行压缩，还可以并行的处理，这样就可以大大的提高 job 执行的速度了。

```
set hive.exec.compress.output=true;
set mapreduce.output.fileoutputformat.compress=true;
set mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.GzipCodec;
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
```

**Hadoop集群支持的压缩算法：**

```
org.apache.hadoop.io.compress.DefaultCodec
org.apache.hadoop.io.compress.GzipCodec
org.apache.hadoop.io.compress.BZip2Codec
org.apache.hadoop.io.compress.DeflateCodec
org.apache.hadoop.io.compress.SnappyCodec
org.apache.hadoop.io.compress.Lz4Codec
com.hadoop.compression.lzo.LzoCodec
com.hadoop.compression.lzo.LzopCodec
```

# 3、Hive架构层面

## 3.1 启用本地抓取（默认开启）

Hive 的某些 SQL 语句需要转换成 MapReduce 的操作，某些 SQL 语句就不需要转换成 MapReduce 操作，但是同学们需要注意，理论上来说，所有的 SQL 语句都需要转换成 MapReduce 操作，只不过Hive 在转换 SQL 语句的过程中会做部分优化，使某些简单的操作不再需要转换成 MapReduce，例如：

1、只是 select * 的时候
2、where 条件针对分区字段进行筛选过滤时
3、带有 limit 分支语句时

## 3.2 本地执行优化

Hive 在集群上查询时，默认是在集群上多台机器上运行，需要多个机器进行协调运行，这种方式很好的解决了大数据量的查询问题。但是在 Hive 查询处理的数据量比较小的时候，其实没有必要启动分布式模式去执行，因为以分布式方式执行设计到跨网络传输、多节点协调等，并且消耗资源。对于小数据集，可以通过本地模式，在单台机器上处理所有任务，执行时间明显被缩短。

三个参数：

```
## 打开hive自动判断是否启动本地模式的开关set hive.exec.mode.local.auto=true;## map任务数最大值，不启用本地模式的task最大个数set hive.exec.mode.local.auto.input.files.max=4;## map输入文件最大大小，不启动本地模式的最大输入文件大小set hive.exec.mode.local.auto.inputbytes.max=134217728;
```

## 3.3 JVM重用

Hive 语句最终会转换为一系列的 MapReduce 任务，每一个MapReduce 任务是由一系列的 MapTask和 ReduceTask 组成的，默认情况下，MapReduce 中一个 MapTask 或者 ReduceTask 就会启动一个JVM 进程，一个 Task 执行完毕后，JVM 进程就会退出。这样如果任务花费时间很短，又要多次启动JVM 的情况下，JVM 的启动时间会变成一个比较大的消耗，这时，可以通过重用 JVM 来解决

JVM也是有缺点的，开启JVM重用会一直占用使用到的 task 的插槽，以便进行重用，直到任务完成后才会释放。如果某个 不平衡的job 中有几个 reduce task 执行的时间要比其他的 reduce task 消耗的时间要多得多的话，那么保留的插槽就会一直空闲却无法被其他的 job 使用，直到所有的 task 都结束了才会释放。

根据经验，一般来说可以使用一个 cpu core 启动一个 JVM，假如服务器有 16 个 cpu core ，但是这个节点，可能会启动 32 个mapTask，完全可以考虑：启动一个JVM，执行两个Task

## 3.4 并行执行

有的查询语句，Hive 会将其转化为一个或多个阶段，包括：MapReduce 阶段、抽样阶段、合并阶段、limit 阶段等。默认情况下，一次只执行一个阶段。但是，如果某些阶段不是互相依赖，是可以并行执行的。多阶段并行是比较耗系统资源的。

一个 Hive SQL 语句可能会转为多个 MapReduce Job，每一个 job 就是一个 stage，这些 Job 顺序执行，这个在 cli 的运行日志中也可以看到。但是有时候这些任务之间并不是是相互依赖的，如果集群资源允许的话，可以让多个并不相互依赖 stage 并发执行，这样就节约了时间，提高了执行速度，但是如果集群资源匮乏时，启用并行化反倒是会导致各个 Job 相互抢占资源而导致整体执行性能的下降。启用并行化

```
## 可以开启并行执行。set hive.exec.parallel=true;## 同一个sql允许最大并行度，默认为8。set hive.exec.parallel.thread.number=16;
```

## 3.5 推测执行

在分布式集群环境下，因为程序Bug（包括Hadoop本身的bug），负载不均衡或者资源分布不均等原因，会造成同一个作业的多个任务之间运行速度不一致，有些任务的运行速度可能明显慢于其他任务（比如一个作业的某个任务进度只有50%，而其他所有任务已经运行完毕），则这些任务会拖慢作业的整体执行进度。为了避免这种情况发生，Hadoop采用了推测执行（Speculative Execution）机制，它根据一定的法则推测出“拖后腿”的任务，并为这样的任务启动一个备份任务，让该任务与原始任务同时处理同一份数据，并最终选用最先成功运行完成任务的计算结果作为最终结果

```
# 启动mapper阶段的推测执行机制set mapreduce.map.speculative=true;# 启动reducer阶段的推测执行机制set mapreduce.reduce.speculative=true;
```

如果用户对于运行时的偏差非常敏感的话，那么可以将这些功能关闭掉。如果用户因为输入数据量很大而需要执行长时间的MapTask或者ReduceTask的话，那么启动推测执行造成的浪费是非常巨大大。其实我一般不使用

## 3.6 Hive严格模式

所谓严格模式，就是强制不允许用户执行有风险的 HiveQL 语句，一旦执行会直接失败。但是Hive中为了提高SQL语句的执行效率，可以设置严格模式，充分利用Hive的某些特点。

```
## 设置Hive的严格模式set hive.mapred.mode=strict;set hive.exec.dynamic.partition.mode=nostrict;
```

注意：当设置严格模式之后，会有如下限制：

```
1、对于分区表，必须添加where对于分区字段的条件过滤select * from student_ptn where age > 25
2、order by语句必须包含limit输出限制select * from student order by age limit 100;
3、限制执行笛卡尔积的查询select a.*, b.* from a, b;
4、在hive的动态分区模式下，如果为严格模式，则必须需要一个分区列式静态分区
```

# 4、数据倾斜

## 4.1 不同数据类型关联产生数据倾斜

```
select * from users aleft outer join logs bon a.usr_id = cast(b.user_id as string)
```

## 4.2 空值过滤

在生产环境经常会用大量空值数据进入到一个reduce中去，导致数据倾斜。

解决办法：

自定义分区，将为空的key转变为字符串加随机数或纯随机数，将因空值而造成倾斜的数据分不到多个Reducer。

注意：对于异常值如果不需要的话，最好是提前在where条件里过滤掉，这样可以使计算量大大减少

## 4.3 group by

采用sum() group by的方式来替换count(distinct)完成计算。

## 4.4 map join

以上讲过了

## 4.5 开启数据倾斜时负载均衡

以上也讲过了

# 5、调优方案

## 5.1 日志表和用户表做链接

```
select * from log a left outer join users b on a.user_id = b.user_id;
```

users 表有 600w+ （假设有5G）的记录，把 users 分发到所有的 map 上也是个不小的开销，而且MapJoin 不支持这么大的小表。如果用普通的 join，又会碰到数据倾斜的问题。

**改进方案：**

```
select /*+mapjoin(x)*/ * from log aleft outer join (
  select /*+mapjoin(c)*/ d.*
   from ( select distinct user_id from log ) c join users d on c.user_id =d.user_id) xon a.user_id = x.user_id;
```

假如，log 里 user_id 有上百万个，这就又回到原来 MapJoin 问题。所幸，每日的会员 uv 不会太多，有交易的会员不会太多，有点击的会员不会太多，有佣金的会员不会太多等等。所以这个方法能解决很多场景下的数据倾斜问题。

## 5.2 位图法求连续七天发朋友圈的用户

每天都要求 微信朋友圈 过去连续7天都发了朋友圈的小伙伴有哪些？假设每个用户每发一次朋友圈都记录了一条日志。每一条朋友圈包含的内容

```
日期，用户ID，朋友圈内容.....dt, userid, content, .....
```

如果 微信朋友圈的 日志数据，按照日期做了分区。

2020-07-06 file1.log(可能会非常大)
2020-07-05 file2.log
…

解决方案：

```
假设微信有10E用户，我们每天生成一个长度为10E的二进制数组，每个位置要么是0，要么是1，如果为1，代表该用户当天发了朋友圈。如果为0，代表没有发朋友圈。
然后每天：10E / 8 / 1024 / 1024 = 119M左右
求Join实现：两个数组做 求且、求或、异或、求反、求新增
```





实际用例: 

```
set hive.exec.dynamic.partition.mode=nonstrict;
set mapred.job.map.capacity=2000;
set hive.exec.parallel=true; # 并行执行
set hive.exec.parallel.thread.number=16;
set mapred.max.split.size=256000000;
set mapred.min.split.size.per.node=100000000;
set mapred.min.split.size.per.rack=100000000;
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
set hive.merge.size.per.task=128000000;
set hive.merge.smallfiles.avgsize=128000000;
```



