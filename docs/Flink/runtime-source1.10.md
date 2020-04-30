| 包名             | 功能描述                                                     | 重点                                                  | 重要性 |
| ---------------- | ------------------------------------------------------------ | ------ | ------ |
| accumulators     | 用户自定义累加器,映射表,传输容器                             |                              |        |
| akka             | akka协议拓展的监管策略,在actor kill时,debug出停止日志        |     |        |
| blob             | 持久(高可用的即各个节点都有的,比如任务jar)/临时(非高可用,单个节点上的)的大文件上传下载的server和client(找文件先从本地找,没有从HA store下载,底层是netty的加密套接字协议实现的),作用比如job jar/graph/配置文件上传,tm日志文件下载,出现过blobserver的异常 | ssl,netty |        |
| broadcast        | BroadcastVariableManager用于管理广播变量的具体化。物化的引用广播变量缓存并在并行子任务之间共享。保留一个引用计数来跟踪是否物料可能会被清理掉。主要用于批/迭代任务,和BroadcastConnectedStream没关系 |  |        |
| checkpoint       | **检查点**                                                   |                                                        | √      |
| client           | 客户端工具类,主要是上传jar/文件(调blob包),还包含job的各种异常,简单的job执行状态 |  |        |
| clusterframework | 集群框架,包含启动jm/tm actor的工具,**taskexecutor的内存配置管理**,应用而非job的状态,资源预算管理器,申请id,资源id,slotid,任务执行的抽象容器(包含拷贝进来的配置文件,各种系统和flink的环境变量),Mesos特有(通过overlays包中的将容器内配置覆盖掉) |  | √      |
| concurrent       | 适配器模式实现的调度Executor,ScheduledExecutor,还有封装ExecutorService的直接执行任务的实现,akka调度的适配器,用于错误恢复调度的工具类FutureUtils | 适配器模式,akka |        |
| deployment       | 部署描述符和其工厂类,Task部署描述符包含输入口描述符集合和结果分区描述符集合,被用于ExecutionGraph |  |        |
| dispatcher       | **Dispatcher组件负责接收作业提交、持久化它们、生成jobmanager来执行作业，并在master出现故障时恢复它们。此外，它还知道Flink session cluster的状态。** |                                                              | √ |
| entrypoint       | 任务进入点这个类的特殊实现可以用于各种集群的会话模式和作业模式,还包含将命令行解析为配置的工具,另外还有些检索器和 在一个进程中同时启动 Dispatcher,ResourceManager,WebMonitorEndpoint的组件 | 模板模式 | √ |
| event       | 运行时taskexecutor交换信息的类型,分runtime事件和task事件,实现io包的IOReadableWritable |                                                              |        |
| execution       | librarycache : 根据jobid和blob key进行任务jar下载,execution state状态机(只有一个图,也是到处用,没有实现统一管理),environment接口,**环境为在任务中执行的代码提供了对任务属性(如名称、并行性)、配置、数据流读取器和写入器以及TaskManager提供的各种组件(如内存管理器、I/O管理器、…** | environment和状态机 |        |
| executiongraph       | **metrics** : 用于未执行时间,执行时间,最后一次重启时间指标的实现;**restart**:重启这个故障恢复策略的细节策略(不,指定失败次数,指定失败率,直接抛异常来验证是否用的是这个的重启策略);**failover**:故障恢复策略(不,全量重启,重启独立task,只适用于task之间没有连接的情况,区域重启 : flip1:任务流水线中的区域重启,需要确定流水线的task与task之间是否有shuffle,没有则可以部分恢复); 包含job,task,分区的信息,job的实际部署节点,边,图,节点的实际实行等;**执行图是协调数据流的分布式执行的中心数据结构。它保持每个并行任务、每个中间流以及它们之间的通信的表示。** |                                                              | √ |
| filecache       | 当任务部署时,FileCache被用于存取任务注册时的缓存文件.检索到的目录将在{ <system-tmp-dir>/tmp_<jobID>/}中展开。并在任务延迟5秒后取消注册时删除，除非同时有新任务请求该文件。 |                                                              |        |
| heartbeat       | HeartbeatListener:与HeartbeatManager交互的接口,有3个用处,通知心跳时间,接受到来的心跳,等待发送的心跳的回应.分3种:jm,tm,resourcemanager心跳,在不同模块都有实现.;HeartbeatManager:心跳管理可以开启停止对心跳目标资源(有clusterframework包的资源id)的监控器,还可以报告该目标的心跳超时;HeartbeatServices:心跳的服务类;HeartbeatTarget:接受,请求心跳接口 |                                                              |        |
| highavailability       | 相关ha接口;nonha:非ha实现,包含进程中选举,和standalone集群;zookeeper:zk ha实现;HighAvailabilityServices接口包含涉及HA的各种服务 | zk实现ha和几种ha的实现方式 | √      |
| history       | FsJobArchivist : 一个将给定文件写json到文件系统并读取回来的工具类,被用于HistoryServer(一般没有):HistoryServer定期检查一组由 FsJobArchivist创建的作业归档目录,并将这些缓存到一个本地目录中。HistoryServer提供了一个WebInterface和REST API来检索关于其已完成作业的信息作业管理器可能已经关闭了。 |                                                              |        |
| instance       | HardwareDescription:硬件描述描述任务管理器可用的资源(cpu核,jvm堆内存,系统管理的内存(用于缓存、散列、排序的内存大小...),物理内存),另外还包含slot共享单元ID,slot共享单元定义了不同的task(来自不同的作业顶点)一起部署在一个slot内。这是一种软许可，而不是硬约束由位置提示定义。 |                                                              |        |
| io       | compression:lz4格式压缩/解压缩;disk:各种查数据的视图层,溢出buff,iomanager:文件channel的读写视图,异步块/批块读写器,异步缓存文件读写器; |                                                              | √ |
| iterative       |  |                                                              |        |
| jobgraph       |                                                              |                                                              |        |
| jobmanager       |                                                              |                                                              |        |
| jobmaster       |                                                              |                                                              |        |
| leaderelection       |                                                              |                                                              |        |
| leaderretrieval       |                                                              |                                                              |        |
| memory       |                                                              |                                                              |        |
| messages       |                                                              |                                                              |        |
| metrics       |                                                              |                                                              |        |
| minicluster       |                                                              |                                                              |        |
| net       |                                                              |                                                              |        |
| operators       |                                                              |                                                              |        |
| plugable       |                                                              |                                                              |        |
| query       |                                                              |                                                              |        |
| registration       |                                                              |                                                              |        |
| resourcemanager       |                                                              |                                                              |        |
| rest       |                                                              |                                                              |        |
| rpc       |                                                              |                                                              |        |
| scheduler       |                                                              |                                                              |        |
| security       |                                                              |                                                              |        |
| shuffle       |                                                              |                                                              |        |
| state       |                                                              |                                                              |        |
| taskexecutor       |                                                              |                                                              |        |
| taskmanager       |                                                              |                                                              |        |
| throwable       |                                                              |                                                              |        |
| topology       |                                                              |                                                              |        |
| types       |                                                              |                                                              |        |
| util       |                                                              |                                                              |        |
| webmonitor       |                                                              |                                                              |        |
| zookeeper       |                                                              |                                                              |        |



