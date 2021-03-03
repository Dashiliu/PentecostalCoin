# 小文件合并

## 小文件带来的问题

1. HDFS的文件元信息，包括位置、大小、分块信息等，都是保存在NameNode的内存中的。每个对象大约占用150个字节，因此一千万个文件及分块就会占用约3G的内存空间，一旦接近这个量级，NameNode的性能就会开始下降了
2. HDFS读写小文件时也会更加耗时，因为每次都需要从NameNode获取元信息，并与对应的DataNode建立连接
3. 对于MapReduce程序来说，小文件还会增加Mapper的个数，每个脚本只处理很少的数据，浪费了大量的调度时间。当然，这个问题可以通过使用CombinedInputFile和JVM重用来解决

## Hive小文件产生的原因

1. kafka 落 hdfs 时刷出频率过高或异常导致小文件,或topic过小,分区数过多导致小文件
2. map 或 reduce 导致小文件过多

## 解决小文件的问题：

1. 落文件程序修改落根据topic大小,文件大小配置并发度.
2. 输入合并。即在Map前合并小文件
3. 输出合并。即在输出结果的时候合并小文件

### 已经有的进行insert overwrite + 计算过程中参数配置

在Hive中使用INSERT OVERWRITE时，
数据会先被写入到数据文件夹的临时文件内，类似于 .hive-staging_hive_ 开头的文件
然后删除所有原文件，将临时文件重命名为”原文件“

### 计算过程中参数配置

**配置Map输入合并**

-- 每个Map最大输入大小，决定合并后的文件数
**set** mapred. max .split.size=256000000;
-- 一个节点上split的至少的大小 ，决定了多个data node上的文件是否需要合并

**set hive.merge.size.per.task = 256\*1000\*1000 ##合并文件的大小**

**set mapred.max.split.size=256000000; ##每个 Map 最大分割大小**

**set mapred.min.split.size.per.node=1; ##一个节点上 split 的最少值**

**set** mapred. min .split.size.per.node=100000000;
-- 一个交换机下split的至少的大小，决定了多个交换机上的文件是否需要合并
**set** mapred. min .split.size.per.rack=100000000;
-- 执行Map前进行小文件合并
**set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; **

-- JVM重用可以使得JVM实例在同一个job中重新使用N次

**set mapreduce.job.jvm.numtasks=1;**

**配置Hive结果合并**

我们可以通过一些配置项来使Hive在执行结束后对结果文件进行合并：

hive.merge.mapfiles 在map-only job后合并文件，默认true

hive.merge.mapredfiles 在map-reduce job后合并文件，默认false

hive.merge.size.per.task 合并后每个文件的大小，默认256000000

hive.merge.smallfiles.avgsize 平均文件大小，是决定是否执行合并操作的阈值，默认16000000

Hive在对结果文件进行合并时会执行一个额外的map-only脚本，mapper的数量是文件总大小除以size.per.task参数所得的值，触发合并的条件是：

根据查询类型不同，相应的mapfiles/mapredfiles参数需要打开；

结果文件的平均大小需要大于avgsize参数的值。

**压缩文件的处理**

对于输出结果为压缩文件形式存储的情况，要解决小文件问题，如果在Map输入前合并，对输出的文件存储格式并没有限制。但是如果使用输出合并，则必须配合SequenceFile来存储，否则无法进行合并，以下是示例：

**set** mapred.output.compression. **type** =BLOCK;
**set** hive.exec.compress.output= **true** ;
**set** mapred.output.compression.codec=org.apache.hadoop.io.compress.LzoCodec;
**set** hive.merge.smallfiles.avgsize=100000000;
**drop**  **table** if **exists** dw_stage.zj_small;
**create**  **table** dw_stage.zj_small
STORED **AS** SEQUENCEFILE
**as**  **select** *
**from** dw_db.dw_soj_imp_dtl
**where** log_dt = '2014-04-14'
**and** paid **like** '%baidu%' ;

**使用HAR归档文件**

Hadoop的归档文件格式也是解决小文件问题的方式之一。而且Hive提供了原生支持：

**set** hive.archive.enabled= **true** ;
**set** hive.archive.har.parentdir.settable= **true** ;
**set** har.partfile.size=1099511627776;
**ALTER**  **TABLE** srcpart ARCHIVE PARTITION(ds= '2008-04-08' , hr= '12' );
**ALTER**  **TABLE** srcpart UNARCHIVE PARTITION(ds= '2008-04-08' , hr= '12' );

如果使用的不是分区表，则可创建成外部表，并使用har://协议来指定路径。

---

**背景**
Hive query将运算好的数据写回hdfs（比如insert into语句），有时候会产生大量的小文件，如果不采用CombineHiveInputFormat就对这些小文件进行操作的话会产生大量的map task，耗费大量集群资源，而且小文件过多会对namenode造成很大压力。所以Hive在正常job执行完之后，会起一个conditional task，来判断是否需要合并小文件，如果满足要求就会另外启动一个map-only job 或者mapred job来完成合并

**参数解释**
hive.mergejob.maponly (默认为true)
如果hadoop版本支持CombineFileInputFormat，则启动Map-only job for merge，否则启动  MapReduce merge job，map端combine file是比较高效的做法

hive.merge.mapfiles(默认为true)
正常的map-only job后，是否启动merge job来合并map端输出的结果

hive.merge.mapredfiles(默认为false)
正常的map-reduce job后，是否启动merge job来合并reduce端输出的结果，建议开启

hive.merge.smallfiles.avgsize(默认为16MB)
如果不是partitioned table的话，输出table文件的平均大小小于这个值，启动merge job，如果是partitioned table，则分别计算每个partition下文件平均大小，只merge平均大小小于这个值的partition。这个值只有当hive.merge.mapfiles或hive.merge.mapredfiles设定为true时，才有效

hive.exec.reducers.bytes.per.reducer(默认为1G)
如果用户不主动设置mapred.reduce.tasks数，则会根据input directory计算出所有读入文件的input summary size，然后除以这个值算出reduce number
  reducers = (int) ((totalInputFileSize + bytesPerReducer - 1) / bytesPerReducer);
  reducers = Math.max(1, reducers);
  reducers = Math.min(maxReducers, reducers);

hive.merge.size.per.task(默认是256MB)
merge job后每个文件的目标大小（targetSize），用之前job输出文件的total size除以这个值，就可以决定merge job的reduce数目。merge job的map端相当于identity map，然后shuffle到reduce，每个reduce dump一个文件，通过这种方式控制文件的数量和大小

```
MapredWork work = (MapredWork) mrTask.getWork();
if (work.getNumReduceTasks() > 0) {
   int maxReducers = conf.getIntVar(HiveConf.ConfVars.MAXREDUCERS);
   int reducers = (int) ((totalSize +targetSize - 1) / targetSize);
   reducers = Math.max(1, reducers);
   reducers = Math.min(maxReducers, reducers);
  work.setNumReduceTasks(reducers);
}
```

mapred.max.split.size(默认256MB)
mapred.min.split.size.per.node(默认1 byte)
mapred.min.split.size.per.rack(默认1 byte)
这三个参数CombineFileInputFormat中会使用，Hive默认的InputFormat是CombineHiveInputFormat，里面所有的调用（包括最重要的getSplits和getRecordReader）都会转换成CombineFileInputFormat的调用，所以可以看成是它的一个包装。CombineFileInputFormat 可以将许多小文件合并成一个map的输入，如果文件很大，也可以对大文件进行切分，分成多个map的输入。一个CombineFileSplit对应一个map的输入，包含一组path(hdfs路径list)，startoffset, lengths, locations(文件所在hostname list)mapred.max.split.size是一个split 最大的大小，mapred.min.split.size.per.node是一个节点上（datanode）split至少的大小，mapred.min.split.size.per.rack是同一个交换机(rack locality)下split至少的大小通过这三个数的调节，组成了一串CombineFileSplit用户可以通过增大mapred.max.split.size的值来减少Map Task数量

**结论**
hive 通过上述几个值来控制是否启动merge file job，通常是建议大家都开启，如果是一堆顺序执行的作业链，只有最后一张表需要固化落地，中间表用好就删除的话，可以在最后一个insert into table之前再开启，防止之前的作业也会launch merge job使得作业变慢。

上周还发现目前启动的针对RCFile的Block Merger在某种少见情况下，会生成duplicated files，Hive代码中本身已经考虑到这点，所以会在Merger Task RCFileMergeMapper的JobClose函数中调用Utilities.removeTempOrDuplicateFiles(fs, intermediatePath, dpCtx),  不过不知道为什么没有生效，还会存在重复文件，需要再研究下

Hive是否起merge job是由conditional task在运行时决定的，如果hadoop job或者hive未如预期般执行合并作业，则可以利用github上的file crush工具完成合并，它的原理也是启动一个mapreduce job完成合并，不过目前只支持textfile 和 sequencefile