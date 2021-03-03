# Hadoop 归档 和HIVE 如何使用har 归档 文件

Hadoop archive 唯一的优势可能就是将众多的小文件打包成一个har 文件了，那这个文件就会按照dfs.block.size 的大小进行分块，因为hdfs为每个块的元数据大小大约为150个字节，如果众多小文件的存在（什么是小文件内，就是小于dfs.block.size 大小的文件，这样每个文件就是一个block）占用大量的namenode 堆内存空间，打成har 文件可以大大降低namenode 守护节点的内存压力。但对于MapReduce 来说起不到任何作用，因为har文件就相当一个目录，仍然不能讲小文件合并到一个split中去，一个小文件一个split ，任然是低效的，这里要说一点<<hadoop 权威指南 中文版>>对这个翻译有问题，上面说可以分配到一个split中去，但是低效的。
     既然有优势自然也有劣势，这里不说它的不足之处，仅介绍如果使用har 并在hadoop中更好的使用har 文件

首先 看下面的命令
     hadoop archive -archiveName 20131101.har /user/hadoop/login/201301/01 /user/hadoop/login/201301/01
     我用上面的命令就可以将 /user/hadoop/login/201301/01 目录下的文件打包成一个 20131101.har 的归档文件，但是系统不会自动删除源文件，需要手动删除
     hadoop fs -rmr /user/hadoop/login/201301/01/*.*.* 我是用正则表达式来删除的，大家根据自己的需求删除原始文件

 有人说了，我删了，归档文件存在，源文件不在了，如果要恢复怎么办，这个也困惑了我，hadoop 好像确实也没有提供这样的API 可以 还原成源文件
 功夫不负有心人，其实也很简单,直接从har 文件中 cp出来就可以了。
     hadoop fs -cp /user/hadoop/login/201301/01/20130101.har/*  /user/hadoop/login/201301/01/

那如何在hive 中使用呢，首先看建表 ：
 CREATE EXTERNAL TABLE login_har(
  ldate string,
  ltime string,
  userid int,
  name string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ' '
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://h60:9000/user/hadoop/login/201301/01'
这是正常的文件 建外表 从而可以不损害源文件的情况下 在Hive中查看，外边有啥优点不多说。
如果是har 文件呢?
 CREATE EXTERNAL TABLE login_har(
  ldate string,
  ltime string,
  userid int,
  name string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ' '
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'har://h60:9000/user/hadoop/login/201301/01/20130101.har'

这样就可以实现，但这样不好，为什么不好呢，我只能制定单一的目录，加入我的数据增加了，如何能动态的修改呢？
其实也简单：
 CREATE EXTERNAL TABLE login_har(
  ldate string,
  ltime string,
  userid int,
  name string)
PARTITIONED BY (
  ym string,
  d string)
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY ' '
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://h60:9000/user/hadoop/login'

先对其父目录建表，然后对年月日进行分区PARTITIONED BY 即是进行分区
再手动修改 其动态分区 即可：
     alter table login_har add partition(ym='201301',d='01') LOCATION 'har:///flume/loginlog/201301/01/20130101.har';
标注为红色的 实践中证明 只能 支持select * from 不加条件的查询，意思就是如果hive mapreduce的话，那就无法 通过这种方式
 alter table login_har add partition(ym='201301',d='01') LOCATION 'hdfs://h60:9000/flume/loginlog/201301/01/20130101.har'; 
只能通过下面的方式进行。

这样不是很好，既可以对hive 表进行分区索引，也可以动态增加har 文件 到新的分区中。har包不能一旦建成不能修改，我们可以打小包，建目录的方式进行分而治之，既满足需求也不影响效率。