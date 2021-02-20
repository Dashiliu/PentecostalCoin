# HBase最佳实践－聊聊HBase核心配置参数

把生命浪费在美好事物上(https://sq.163yun.com/user/165274507257987072)2018-06-29 09:43

参数之于软件系统就像按钮之于工程系统，如下图所示（俄罗斯核电站操作台），绝大多数工程师对于工程系统的认知就是首先从这些按钮来的，而且通常来说按钮越多，系统就会越复杂。认知过程无非三个阶段，首先弄明白这些按钮都用来控制神马，再者是在什么场景下需要旋转按钮、如何旋转，最

后就是弄清楚为什么在这种场景下这么旋转，在另外一种场景下那样旋转。软件参数亦是一样。

接下来笔者会将HBase中常见的参数分类整理（版本为HBase 1.1.2），主要解释每个参数的实际意义以及在生产线上的配置注意事项，如果关注这些参数背后的运作原理，可以参考之前的HBase原理相关文章，里面或多或少都有提及。

－－－－－－－－－－－－－－－－－－Region－－－－－－－－－－－－－－－－－－－

**hbase.hregion.max.filesize**：默认10G，简单理解为Region中任意HStore所有文件大小总和大于该值就会进行分裂。

解读：实际生产环境中该值不建议太大，也不能太小。太大会导致系统后台执行compaction消耗大量系统资源，一定程度上影响业务响应；太小会导致Region分裂比较频繁（分裂本身其实对业务读写会有一定影响），另外单个RegionServer中必然存在大量Region，太多Region会消耗大量维护资源，并且在rs下线迁移时比较费劲。综合考虑，建议线上设置为50G～80G左右。

－－－－－－－－－－－－－－－－－－BlockCache－－－－－－－－－－－－－－－－－－－

该模块主要介绍BlockCache相关的参数，这个模块参数非常多，而且比较容易混淆。不同BlockCache策略对应不同的参数，而且这里参数配置会影响到Memstore相关参数的配置。笔者对BlockCache策略一直持有这样的观点：RS内存在20G以内的就选择LRUBlockCache，大于20G的就选择BucketCache中的Offheap模式。接下来所有的相关配置都基于BucketCache的offheap模型进行说明。

**file.block.cache.size**：默认0.4，该值用来设置LRUBlockCache的内存大小，0.4表示JVM内存的40%。

解读：当前HBase系统默认采用LRUBlockCache策略，BlockCache大小和Memstore大小均为JVM的40%。但对于BucketCache策略来讲，Cache分为了两层，L1采用LRUBlockCache，主要存储HFile中的元数据Block，L2采用BucketCache，主要存储业务数据Block。因为只用来存储元数据Block，所以只需要设置很小的Cache即可。建议线上设置为0.05～0.1左右。

**hbase.bucketcache.ioengine**：offheap

**hbase.bucketcache.size**：堆外存大小，设置多大就看自己的物理内存大小喽

－－－－－－－－－－－－－－－－－－Memstore－－－－－－－－－－－－－－－－－－－

**hbase.hregion.memstore.flush.size**：默认128M（134217728），memstore大于该阈值就会触发flush。如果当前系统flush比较频繁，并且内存资源比较充足，可以适当将该值调整为256M。

**hbase.hregion.memstore.block.multiplier**：默认4，表示一旦某region中所有写入memstore的数据大小总和达到或超过阈值hbase.hregion.memstore.block.multiplier * hbase.hregion.memstore.flush.size，就会执行flush操作，并抛出RegionTooBusyException异常。

解读：该值在1.x版本默认值为2，比较小，为了保险起见会将其修改为5。而当前1.x版本默认值已经为4，通常不会有任何问题。如果日志中出现类似”Above memstore limit, regionName = ***, server=***,memstoreSizse=***,blockingMemstoreSize=***”，就需要考虑修改该参数了。

**hbase.regionserver.global.memstore.size**：默认0.4，表示整个RegionServer上所有写入memstore的数据大小总和不能超过该阈值，否则会阻塞所有写入请求并强制执行flush操作，直至总memstore数据大小降到hbase.regionserver.global.memstore.lowerLimit以下。

解读：该值在offheap模式下需要配置为0.6～0.65。一旦写入出现阻塞，立马查看日志定位“Blocking update on ***: the global memstore size *** is >= than blocking *** size”。一般情况下不会出现这类异常，如果出现需要明确是不是region数目太多、单表列族设计太多。

**hbase.regionserver.global.memstore.lowerLimit**：默认0.95。不需要修改。

**hbase.regionserver.optionalcacheflushinterval**：默认1h（3600000），hbase会起一个线程定期flush所有memstore，时间间隔就是该值配置。

解读：生产线上该值建议设大，比如10h。因为很多场景下1小时flush一次会导致产生很多小文件，一方面导致flush比较频繁，一方面导致小文件很多，影响随机读性能。因此建议设置较大值。

－－－－－－－－－－－－－－－－－－Compaction－－－－－－－－－－－－－－－－－－－

compaction模块主要用来合并小文件，删除过期数据、deleted数据等。涉及参数较多，对于系统读写性能影响也很重要，下面主要介绍部分比较核心的参数。

**hbase.hstore.compactionThreshold：**默认为3，compaction的触发条件之一，当store中文件数超过该阈值就会触发compaction。通常建议生产线上写入qps较高的系统调高该值，比如5～10之间。

**hbase.hstore.compaction.max：**默认为10，最多可以参与minor compaction的文件数。该值通常设置为**hbase.hstore.compactionThreshold略的2～3倍。**

**hbase.regionserver.thread.compaction.throttle：**默认为2G，评估单个compaction为small或者large的判断依据。为了防止large compaction长时间执行阻塞其他small compaction，hbase将这两种compaction进行了分离处理，每种compaction会分配独立的线程池。

**hbase.regionserver.thread.compaction.large/small：**默认为1，large和small compaction的处理线程数。生产线上建议设置为5，强烈不建议再调太大（比如10），否则会出现性能下降问题。

**hbase.hstore.blockingStoreFiles：**默认为10，表示一旦某个store中文件数大于该阈值，就会导致所有更新阻塞。生产线上建议设置该值为100，避免出现阻塞更新，一旦发现日志中打印’*** too many store files***’，就要查看该值是否设置正确。

**hbase.hregion.majorcompaction：**默认为1周（1000*60*60*24*7），表示major compaction的触发周期。生产线上建议大表major compaction手动执行，需要将此参数设置为0，即关闭自动触发机制。

－－－－－－－－－－－－－－－－－－HLog－－－－－－－－－－－－－－－－－－－

**hbase.regionserver.maxlogs**：默认为32，region flush的触发条件之一，wal日志文件总数超过该阈值就会强制执行flush操作。该默认值对于很多集群来说太小，生产线上具体设置参考HBASE-14951

**hbase.regionserver.hlog.splitlog.writer.threads**：默认为3，regionserver恢复数据时日志按照region切分之后写入buffer，重新写入hdfs的线程数。生产环境因为region个数普遍较多，为了加速数据恢复，建议设置为10。

－－－－－－－－－－－－－－－－－－Call Queue－－－－－－－－－－－－－－－－－－－

**hbase.regionserver.handler.count**：默认为30，服务器端用来处理用户请求的线程数。生产线上通常需要将该值调到100～200。

解读：response time = queue time + service time，用户关心的请求响应时间由两部分构成，优化系统需要经常关注queue time，如果用户请求排队时间很长，首要关注的问题就是hbase.regionserver.handler.count**是否没有调整。**

**hbase.ipc.server.callqueue.handler.factor :** 默认为0，服务器端设置队列个数，假如该值为0.1，那么服务器就会设置handler.count * 0.1 = 30 * 0.1 = 3个队列。

**hbase.ipc.server.callqueue.read.ratio** : 默认为0，服务器端设置读写业务分别占用的队列百分比以及handler百分比。假如该值为0.5，表示读写各占一半队列，同时各占一半handler。

**hbase.ipc.server.call.queue.scan.ratio**：默认为0，服务器端为了将get和scan隔离设置了该参数。

不得不说，队列隔离的这些参数真心蛋疼！！！

－－－－－－－－－－－－－－－－－－Other－－－－－－－－－－－－－－－－－－－

**hbase.online.schema.update.enable**：默认为false，表示更新表schema的时候不再需要先disable再enable，直接在线更新。该参数在HBase 2.0之后将会默认为true。生产线上建议设置为true。

**hbase.quota.enabled**：默认为false，表示是否开启quota功能，quota功能主要用来限制用户/表的QPS，起到限流作用。生产线上建议设置为true。

**hbase.snapshot.enabled**：默认为false，表示是否开启snapshot功能，snapshot功能主要用来备份HBase数据。生产线上建议设置为true。

**zookeeper.session.timeout**：默认180s，表示zookeeper客户端与服务器端session超时时间，超时之后RegionServer将会被踢出集群。

解读：有两点需要重点关注，其一是该值需要与Zookeeper服务器端session相关参数一同设置才会生效，一味的将该值增大而不修改ZK服务端参数，可能并不会实际生效。其二是通常情况下离线集群可以将该值设置较大，在线业务需要根据业务对延迟的容忍度考虑设置。

**hbase.zookeeper.useMulti**：默认为false，表示是否开启zookeeper的multi-update功能，该功能在某些场景下可以加速批量请求完成，而且可以有效防止部分异常问题。生产线上建议设置为true。注意设置为true的前提是Zookeeper服务端的版本在3.4以上，否则会出现zk客户端夯住的情况。

**hbase.coprocessor.master.classes**：生产线上建议设置org.apache.hadoop.hbase.security.access.AccessController，可以使用grant命令对namespace\table\cf设置访问权限。

**hbase.coprocessor.region.classes**：生产线上建议设置org.apache.hadoop.hbase.security.token.TokenProvider,org.apache.hadoop.hbase.security.access.AccessController，同上。

HBase系统中存在大量参数，大部分参数并不需要修改，只有一小部分核心参数需要根据集群硬件环境、网络环境以及业务类型进行调整设置。本文主要将这些核心参数的意义进行了简单说明并给出了生产线上的调整方案，方便HBase新同学能够更加容易入手（笔者还是建议有兴趣的同学能够了解这些参数背后的故事）。需要注意的是上述参数可能并不完整，比如kerberos、replication、rest、thrift、hdfs client等相关模块参数并没有涉及，如有需要可以参考其他资料进行配置。