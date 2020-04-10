# Apache Phoenix

首先Phoenix是HBase之上的SQL工具，至于HBase是什么，我就不介绍了，你若不懂，就不需要往下继续看了。Phoenix旨在通过标准的SQL语法来简化HBase的使用，并可以使用标准的JDBC连接HBase，而不是通过HBase的Java客户端APIs。它可以让你执行所有的CRUD和DDL操作，比如创建一张表，插入数据以及查询数据。SQL和JDBC可以大大减少用户代码的开发，当然它也提供一些性能优化的手段，通过SQL和JDBC，你可以更方便的将HBase集成到你现有的系统或者工具。

当Phoenix接收到SQL查询后，它会在本地编译成HBase的API，然后推到集群进行分布式的查询或计算。它自动创建了一个元数据库用来存储HBase的表的元数据信息。因为Phoenix是直接调用的HBase的API，coprocessors和自定义的filters，所以对于大量小查询可以实现毫秒级返回，千万级别的数据实现秒级返回。

## 使用场景

Phoenix非常适合HBase的随机访问，它的二级索引特性同时可以让你实现非主键查询的快速返回，而不需要进行全表扫描。它可以让你像传统数据库表的方式创建和管理HBase中的表，同时Phoenix也支持复合主键。

Phoenix可以给Rowkey加盐，从而避免因为简单递增的Rowkey引起的RegionServer热点问题。通过指定不同的租户连接实现数据访问的隔离，从而实现多租户，租户只能访问属于他的数据。

虽然Phoenix有这么多优势，但是它依旧无法替代RDBMS。比如它还有以下限制：

- Phoenix不支持跨行的事务
- 查询优化和join机制比大多数RDBMS要简陋
- 二级索引是通过索引表实现的，主表和索引表的同步会存在问题，虽然只是在一段很短的时间内。所以索引无法完全满足ACID
- 多租户功能比较简单

### 与Hive/Impala/第三方/Region内索引 方案的比较

Hive/Impala也可以作为HBase之上的SQL工具。包括Phoenix这3个工具在很多功能上都有一些重叠，比如它们都提供SQL执行以及JDBC驱动

不像Impala和Hive，Phoenix与HBase结合更加紧密，从而可以更好的利用HBase的一些特性，比如coprocessors和skip scans。

- Phoenix的目标是在HBase之上提供一个高效的类关系型数据库的工具，定位为低延时的查询应用。Impala则主要是基于HDFS的一些主流文件格式如文本或Parquet提供探索式的交互式查询。Hive类似于数据仓库，定位为需要长时间运行的批作业。
- Phoenix很适合需要在HBase之上使用SQL实现CRUD，Impala则适合Ad-hoc的分析类工作负载，Hive则适合批处理如ETL。
- Phoenix非常轻量级，因为它不需要额外的服务。
- Phoenix还支持一些高级功能，比如多个二级索引，flashback查询等。无论是Impala还是Hive都无法提供二级索引支持。

以下是比较：

|                 | Apache Phoenix                           | Impala                       | Hive          | 第三方                         | Region内方案(如Pharos)                        |
| --------------- | ---------------------------------------- | ---------------------------- | ------------- | ------------------------------ | --------------------------------------------- |
| 语法            | SQL                                      | SQL                          | HiveQL        | 基于slor/ES可实现标签化查询    | 索引是自己需要提前订好的,所以语法可以自己实现 |
| 定位            | 为低延时应用在HBase之上提供高效的SQL查询 | 大数据集之上的交互式探索分析 | 批处理比如ETL | 基于HBase的查询服务,云厂商方案 | 可插拔查询方案                                |
| 优点            | 简单方便                                 | -                            | 集成HIve      | 标签化,更快,可使用ES的优势     | 索引利用HBase rowkey机制                      |
| 缺点            | 安装需要重启,历史表需要映射              | -                            | -             | 架构复杂,成本高                | 对原表有很大影响                              |
| 二级索引        | Yes(无法保证ACID)                        | No                           | No            | Yes                            | Yes(ACID实现不完善,只是思路)                  |
| 额外的服务      | No                                       | Yes                          | Yes           | Yes                            | NO                                            |
| HBase的高级特性 | Yes                                      | No                           | No            | Yes                            | Yes                                           |

## phoenix 安装

安装(jar包放到每个HBase RS lib下) -> 配置(修改HBase 配置启用2级索引) -> 重启HBase RS

```
<property>
     <name>hbase.regionserver.wal.codec</name>
     <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
<property>
     <name>hbase.master.loadbalancer.class</name>
     <value>org.apache.phoenix.hbase.index.balancer.IndexLoadBalancer</value>
</property>
<property>
     <name>hbase.coprocessor.master.classes</name>
     <value>org.apache.phoenix.hbase.index.master.IndexMasterObserver</value>
</property>
```

## phoenix 常用操作

**语法**   :   http://phoenix.apache.org/language/index.html#close 

- **phoenix-sqlline.py** 是执行SQL的命令脚本，在执行该命令之前，你需要指定HBase集群的Zookeeper地址，比如：phoenix-sqlline.py zk01.example.com:2181。如果想要在命令行执行一个SQL文件，可以在命令行直接带上该文件。比如：sqlline.py zk01.example.com:2181 sql_queries.sql
- **phoenix-psql.py** 可以用来加载CSV数据，后面也可以直接带上需要执行的SQL脚本，也可以不加。比如：phoenix-psql.py zk01.example.com:2181 create_stmts.sql data.csvsql_queries.sql
- **phoenix-performance.py** 是性能测试脚本 可以用于一次插入多少行数据。比如：phoenix-psql.py zk01.example.com:2181 100000

### 表创建

```
!help
!quit
!dbinfo
!大小写问题,小写sql不强转用"" 例如select  * from "hz"."test_table";
create table test (id varchar primary key,name varchar,age integer ); 

效果:
HBase => 'TEST', {TABLE_ATTRIBUTES => {coprocessor$1 => '|org.apache.phoenix.coprocessor.ScanRegionObserver|805306366|', coprocessor$2 => '|org.apache.phoenix.coprocessor.UngroupedAggregateRegionObserver|805306366|', coprocessor$3 => '|org.apache.phoenix.coprocessor.GroupedAggregateRegionObserver|805306366|', coprocessor$4 => '|org.apache.phoenix.coprocessor.ServerCachingEndpointImpl|805306366|', coprocessor$5 => '|org.apache.phoenix.hbase.index.Indexer|805306366|org.apache.hadoop.hbase.index.codec.class=org.apache.phoenix.index.PhoenixIndexCodec,index.builder=org.apache.phoenix.index.PhoenixIndexBuilder'}, {NAME => '0', BLOOMFILTER => 'NONE', DATA_BLOCK_ENCODING => 'FAST_DIFF'}

phoenix => TEST TABLE

upsert into test(id,name,age) values('000001','liubei',43);
select * from  test;
upsert into test(id,name,age) values('000001','liubei',42);
select * from  test;
已修改
drop table test;

效果:
HBase => 没有
phoenix => 没有
```

### Hbase 表映射到phoenix 视图

```
HBase shell 创建表：
create 'USER', 'INFO'

效果:
HBase => 'USER', {NAME => 'INFO'}
phoenix => 没有

put 'USER', 'bbbZZZ1004', 'INFO:NAME', 'WeiDong'
put 'USER', 'bbbZZZ1004', 'INFO:AGE', '27'
put 'USER', 'bbbZZZ1004', 'INFO:HOME', 'ZhouKou'
put 'USER', 'bbbZZZ1005', 'INFO:NAME', 'XiaoMeng'
put 'USER', 'bbbZZZ1005', 'INFO:AGE', '27'
put 'USER', 'bbbZZZ1005', 'INFO:HOME', 'YuLin'
put 'USER', 'bbbTTT1006', 'INFO:NAME', 'GuiPing'
put 'USER', 'bbbTTT1006', 'INFO:AGE', '39'
put 'USER', 'bbbTTT1006', 'INFO:HOME', 'ZhengZhou'

scan 'USER', {LIMIT=>10}

#phoenix 创建原HBase表视图
create view USER (
    pk varchar primary key,
    info.name varchar,
    info.age  varchar,
    info.home varchar
) as select * from USER;

phoenix => USER  VIEW
HBase =>  'USER', {TABLE_ATTRIBUTES => {coprocessor$1 => '|org.apache.phoenix.coprocessor.ScanRegionObserver|805306366|', coprocessor$2 => '|org.apache.phoenix.coprocessor.UngroupedAggregateRegionObserver|805306366|', coprocessor$3 => '|org.apache.phoenix.coprocessor.GroupedAggregateRegionObserver|805306366|', coprocessor$4 => '|org.apache.phoenix.coprocessor.ServerCachingEndpointImpl|805306366|'}, {NAME => 'INFO'}

drop view USER;
phoenix => 没有
HBase => 'USER', {TABLE_ATTRIBUTES => {coprocessor$1 => '|org.apache.phoenix.coprocessor.ScanRegionObserver|805306366|', coprocessor$2 => '|org.apache.phoenix.coprocessor.UngroupedAggregateRegionObserver|805306366|', coprocessor$3 => '|org.apache.phoenix.coprocessor.GroupedAggregateRegionObserver|805306366|', coprocessor$4 => '|org.apache.phoenix.coprocessor.ServerCachingEndpointImpl|805306366|'}, {NAME => 'INFO'}

!tables
!describe user
select * from user;
put 'USER', 'cccTTT1007', 'INFO:NAME', 'YongFa'
put 'USER', 'cccTTT1007', 'INFO:AGE', '27'
put 'USER', 'cccTTT1007', 'INFO:HOME', 'LuYi'
put 'USER', 'cccTTT1008', 'INFO:NAME', 'LuiYa'
put 'USER', 'cccTTT1008', 'INFO:AGE', '27'
put 'USER', 'cccTTT1008', 'INFO:HOME', 'ZhengZhou'
put 'USER', 'cccSSS1009', 'INFO:NAME', 'KeMeng'
put 'USER', 'cccSSS1009', 'INFO:AGE', '27'
put 'USER', 'cccSSS1009', 'INFO:HOME', 'NanYang'
select * from user;
```

### 添加二级索引

```rust
测试用例 先建表:
create table hbase_test
(
s1 varchar not null primary key,
s2 varchar,
s3 varchar,
s4 varchar,
s5 varchar,
s6 varchar,
s7 varchar,
s8 varchar,
s9 varchar,
s10 varchar,
s11 varchar
);
bulkload导入数据:
准备:
hbase_data.csv
340111200507061443,鱼言思,0,遂宁,国家机关,13004386766,15900042793,广州银行1,市场三街65号-10-8,0,1
320404198104281395,暨梅爱,1,临沧,服务性工作人员,13707243562,15004903315,广州银行1,太平角六街145号-9-5,0,3
371326195008072277,人才奇,1,黔西南,办事人员和有关人员,13005470170,13401784500,广州银行1,金湖大厦137号-5-5,1,0
621227199610189727,谷岚,0,文山,党群组织,13908308771,13205463874,广州银行1,仰口街21号-6-2,1,3
533324200712132678,聂健飞,1,辽阳,不便分类的其他劳动者,15707542939,15304228690,广州银行1,郭口东街93号-9-3,0,2
632622196202166031,梁子伯,1,宝鸡,国家机关,13404591160,13503123724,广州银行1,逍遥一街35号-14-8,1,4
440883197110032846,黎泽庆,0,宝鸡,服务性工作人员,13802303663,13304292508,广州银行1,南平广场113号-7-8,1,4
341500196506180162,暨芸贞,0,黔西南,办事人员和有关人员,13607672019,13200965831,广州银行1,莱芜二路117号-18-3,1,4
511524198907202926,滕眉,0,南阳,国家机关,15100215934,13406201558,广州银行1,江西支街52号-10-1,0,3
420205198201217829,陶秀,0,泸州,商业工作人员,13904973527,15602017043,广州银行1,城武支大厦126号-18-2,1,0
hadoop fs -put hbase_data.csv /tmp/jinzl
执行bulkload
HADOOP_CLASSPATH=/opt/cloudera/parcels/CDH/lib/hbase/hbase-protocol-1.2.0-cdh5.13.0.jar:/opt/cloudera/parcels/CDH/lib/hbase/conf hadoop jar /opt/cloudera/parcels/CDH/lib/hbase/lib/phoenix-4.14.1-HBase-1.1-client.jar org.apache.phoenix.mapreduce.CsvBulkLoadTool -t hbase_test -i /tmp/jinzl/hbase_data.csv


1） 不加排序：Create INDEX 索引名 ON 表名（列名A，列表B***） 
2） 加排序：Create INDEX 索引名 ON 表名（列名A DESC，列表B***）
举例如下：
create INDEX id_idx on tower_info("tower_id" ASC ,"create_time"  DESC ,"system","sub_system"）
    
create INDEX id_idx on test(id ASC ,name  DESC ,age);
# 常用删除
drop TABLE if EXISTS TOWER_INFO（表名）
drop index  TOWER_IDX（索引名） ON TOWER_INFO（表名）；
DROP SEQUENCE IF EXISTS test_sequence
```

###  自增主键

```csharp
create table test ("id" BIGINT not null primary key,"tower_id" integer(11)）;
                   
CREATE SEQUENCE test_sequence  START WITH 10000 INCREMENT BY 1 CACHE 1000;

# 创建自增序列说明如下：

CREATE SEQUENCE [IF NOT EXISTS] SCHEMA.SEQUENCE_NAME
[START WITH number]
[INCREMENT BY number]
[MINVALUE number]
[MAXVALUE number]
[CYCLE]
[CACHE number]

参数说明：
sqe_name:序列名
increment:可选子句，表示序列的增量，正数表示生成一个递增的序列，负数表示生成一个递减的序列，其默认值是1.
minvalue:可选子句，决定序列生成的最小值
maxvalue:可选子句，决定序列生成的最大值
start:可选子句，指定序列的开始位置，默认递增序列的起始值为minvalue,递减序列的起始值为maxvalue.
cache:可选子句，决定是否产生序列号预分配并存储在内存中。
cycle:可选关键字，当序列达到最大值或者最小值时，可以继续复位下去；如果是递增系列达到maxvalue，它将又从minvalue继续递增，如果是递减系列达到minvalue，它将从maxvalue继续递减。如果忽略该关键，当其他达到最大值或者最小时仍继续递增/减时将会返回一个错误。
                   
upsert into test ("id", "tower_id") values (NEXT VALUE FOR test_sequence,100)
DROP SEQUENCE IF EXISTS test_sequence
```

## 数据导入和导出 

### bulkload 导入到HBase

Phoenix提供了批量导入/导出数据的方式。批量导入只支持csv格式，分隔符为逗号。

> ```
> 示例: 
> [ec2-user@ip-172-31-22-86 ~]$ HADOOP_CLASSPATH=/opt/cloudera/parcels/CDH/lib/hbase/hbase-protocol-1.2.0-cdh5.12.1.jar:/opt/cloudera/parcels/CDH/lib/hbase/conf hadoop jar /opt/cloudera/parcels/CLABS_PHOENIX/lib/phoenix/phoenix-4.7.0-clabs-phoenix1.3.0-client.jar org.apache.phoenix.mapreduce.CsvBulkLoadTool -t item -i /fayson/item.csv
> 
> 17/10/03 10:32:24 INFO util.QueryUtil: Creating connection with the jdbc url: jdbc:phoenix:ip-172-31-21-45.ap-southeast-1.compute.internal,ip-172-31-22-86.ap-southeast-1.compute.internal,ip-172-31-26-102.ap-southeast-1.compute.internal:2181:/hbase;
> ...
> 17/10/03 10:32:24 INFO zookeeper.ZooKeeper: Initiating client connection, connectString=ip-172-31-21-45.ap-southeast-1.compute.internal:2181,ip-172-31-22-86.ap-southeast-1.compute.internal:2181,ip-172-31-26-102.ap-southeast-1.compute.internal:2181 sessionTimeout=60000 watcher=hconnection-0x7a9c0c6b0x0, quorum=ip-172-31-21-45.ap-southeast-1.compute.internal:2181,ip-172-31-22-86.ap-southeast-1.compute.internal:2181,ip-172-31-26-102.ap-southeast-1.compute.internal:2181, baseZNode=/hbase
> 17/10/03 10:32:24 INFO zookeeper.ClientCnxn: Opening socket connection to server ip-172-31-21-45.ap-southeast-1.compute.internal/172.31.21.45:2181. Will not attempt to authenticate using SASL (unknown error)
> ...
> 17/10/03 10:32:30 INFO mapreduce.Job: Running job: job_1507035313248_0001
> 17/10/03 10:32:38 INFO mapreduce.Job: Job job_1507035313248_0001 running in uber mode : false
> 17/10/03 10:32:38 INFO mapreduce.Job:  map 0% reduce 0%
> 17/10/03 10:32:52 INFO mapreduce.Job:  map 100% reduce 0%
> 17/10/03 10:33:01 INFO mapreduce.Job:  map 100% reduce 100%
> 17/10/03 10:33:01 INFO mapreduce.Job: Job job_1507035313248_0001 completed successfully
> 17/10/03 10:33:01 INFO mapreduce.Job: Counters: 50
> ...
> 17/10/03 10:33:01 INFO mapreduce.AbstractBulkLoadTool: Loading HFiles from /tmp/fef0045b-8a31-4d95-985a-bee08edf2cf9
> ```

### 使用Phoenix从HBase中导出数据到HDFS

> Phoenix还提供了使用MapReduce导出数据到HDFS的功能，以pig的脚本执行。首先准备pig脚本。
>
> ```
> [ec2-user@ip-172-31-22-86 ~]$ cat export.pig 
> REGISTER /opt/cloudera/parcels/CLABS_PHOENIX/lib/phoenix/phoenix-4.7.0-clabs-phoenix1.3.0-client.jar;
> rows = load 'hbase://query/SELECT * FROM ITEM' USING org.apache.phoenix.pig.PhoenixHBaseLoader('ip-172-31-21-45:2181');
> STORE rows INTO 'fayson1' USING PigStorage(',');
> [ec2-user@ip-172-31-22-86 ~]$
> 
> 
> 执行该脚本
> [ec2-user@ip-172-31-22-86 ~]$ pig -x mapreduce export.pig 
> ...
> Counters:
> Total records written : 102000
> Total bytes written : 4068465
> Spillable Memory Manager spill count : 0
> Total bags proactively spilled: 0
> Total records proactively spilled: 0
> 
> Job DAG:
> job_1507035313248_0002
> 
> 2017-10-03 10:45:38,905 [main] INFO  org.apache.pig.backend.hadoop.executionengine.mapReduceLayer.MapReduceLauncher - Success!
> 
> ```

## 其他问题汇总

1. phoenix 使用rowkey模糊查询效率特别低

2. Phoenix中建立hbase的映射表不只是加载元数据，还会为HBase 中每一条数据增加一空列标识，如果数据量太大，可能导致超时中断。建议先建立好Phoenix映射表，然后加载数据或增加服务端配置，延长服务端超时时间。**(这个测过了,没有改变HBase原始数据文件,且官网也说明了不会改变原始数据,view是只读 Table is read only.)**

3. 异步方式构建索引过程中，出现问题：不识别Phoenix中小写字母表，不知是不是版本低的问题。

4.  创建Phoenix二级索引后，只能通过Phoenix接口加载数据，直接操作hbase无效的，也就是说只能通过jdbc和加载CSV文件方式加载数据。

5. 为已有数据phoenix表补建索引，亦可能导致超时中断。建议建立phoenix-HBase表时即建好索引，再接数据。

6. 日期转换

   ```csharp
   select * from test where "create_time" >= TO_DATE(TO_CHAR(?,'yyyy-MM-dd HH:mm:ss'))
   ```