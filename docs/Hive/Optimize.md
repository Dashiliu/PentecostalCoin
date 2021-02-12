分桶表
CREATE TABLE table_name 
PARTITIONED BY (partition1 data_type, partition2 data_type,….) CLUSTERED BY (column_name1, column_name2, …) 
SORTED BY (column_name [ASC|DESC], …)] 
INTO num_buckets BUCKETS;



序列化 
LazySimpleSerDe 
优化 : OrcSerde 从ods往后,开启向量化,一次处理多行默认1024,而非一次一行

Hive支持TEXTFILE, SEQUENCEFILE, AVRO, RCFILE, ORC,以及PARQUET文件格式，可以通过两种方式指定表的文件格式：

- CREATE TABLE … STORE AS :即在建表时指定文件格式，默认是TEXTFILE
- ALTER TABLE … [PARTITION partition_spec] SET FILEFORMAT :修改具体表的文件格式

如果未指定文件存储格式，则默认使用的是参数**hive.default.fileformat**设定的格式。

如果数据存储在小于块大小的小文件中，则可以使用**SEQUENCE**文件格式。如果要以减少存储空间并提高性能的优化方式存储数据，则可以使用**ORC文件**格式，而当列中嵌套的数据过多时，**Parquet**格式会很有用。因此，需要根据拥有的数据确定输入文件格式。



每种格式的简单区别

压缩格式的简单区别

![hive orc](Optimize.assets/20160401-11.jpg)

上图中原始的TEXT文本文件为585GB，使用Hive早期的RCFILE压缩后为505GB，使用Impala中的PARQUET压缩后为221GB，而Hive中的ORC压缩后仅为131GB，压缩比最高。

org.apache.hadoop.io.compress.DefaultCodec
org.apache.hadoop.io.compress.GzipCodec
org.apache.hadoop.io.compress.BZip2Codec
com.hadoop.compression.lzo.LzopCodec
org.apache.hadoop.io.compress.Lz4Codec
org.apache.hadoop.io.compress.SnappyCodec







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



https://mp.weixin.qq.com/s/-xIKcMhyMw2hluzF4EQLGg

hive.exec.mode.local.auto=false # 本地模式 : 消除了提交到集群的overhead，所以比较适合数据量很小，且逻辑不复杂的任务,mapper数必须小于hive.exec.mode.local.auto.tasks.max（默认值4），reducer数必须为0或1，才会真正用本地模式执行
hive.exec.mode.local.auto.input.files.max=4
hive.exec.mode.local.auto.inputbytes.max=134217728

---

##### **select型**

设置hive.fetch.task.conversion=none会以集群模式运行，无论是否有limit。在数据量小时建议使用hive.fetch.task.conversion=more,此时select配合limit以单机执行获取样本数据，执行更快

常见的select配合order by/group by等基本操作不在此赘述

注：
select查询可以通过split.maxsize和split.minsize控制并发MAPPER数量

hive.fetch.task.conversion=minimal
hive.fetch.task.conversion.threshold=268435456

##### **insert型**

分为两种

- insert into
- insert overwrite

配合分区可以达到重写分区或者在分区追加数据的目的。还可以配合动态分区模式插入对应分区

开启动态分区:

```
// 开启动态分区模式set hive.exec.dynamic.partition=true;// 开启动态分区非严格模式（多分区时首分区支持动态分区必要条件，首分区为静态分区可以不设置）set hive.exec.dynamic.partition.mode=nonstrict;// 单节点上限set hive.exec.max.dynamic.partitions.pernode=100;// 集群上限set hive.exec.max.dynamic.partitions=1000;
```

开启之后可以利用SQL达到动态插入的目的:

```
// 根据分区day动态插入数据insert into table default.test partition(day) select id,day from orginal
```

---

几种join 几种sortby 用法

数据模型?

ch的总结 <- 大大加分项 sql部分加强
