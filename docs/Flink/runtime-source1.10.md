| 包名             | 功能描述                                                     | 技术点                                                  | 重要性 |
| ---------------- | ------------------------------------------------------------ | ------ | ------ |
| accumulators     | 用户自定义累加器,映射表,传输容器                             |                              |        |
| akka             | akka协议拓展的监管策略,在actor kill时,debug出停止日志        |     |        |
| blob             | 持久(高可用的即各个节点都有的,比如任务jar)/临时(非高可用,单个节点上的)的大文件上传下载的server和client(找文件先从本地找,然后..,底层是netty的加密套接字协议实现的),作用比如job jar/graph/配置文件上传,tm日志文件下载,出现过blobserver的异常 | ssl,netty |        |
| broadcast        | BroadcastVariableManager用于管理广播变量的具体化。物化的引用广播变量缓存并在并行子任务之间共享。保留一个引用计数来跟踪是否物料可能会被清理掉。主要用于批/迭代任务,和BroadcastConnectedStream没关系 |  |        |
| checkpoint       | 检查点                                                       |                                                        | √      |
| client           | 客户端工具类,主要是上传jar/文件(调blob包),还包含job的各种异常,简单的job执行状态 |  |        |
| clusterframework | 集群框架,包含启动jm/tm actor的工具,taskexecutor的内存配置管理,应用而非job的状态,资源预算管理器,申请id,资源id,slotid,任务执行的抽象容器(包含拷贝进来的配置文件,各种系统和flink的环境变量),Mesos特有(通过overlays包中的将容器内配置覆盖掉) |  | √      |
| concurrent       | 适配器模式实现的调度Executor,ScheduledExecutor,还有封装ExecutorService的直接执行任务的实现,akka调度的适配器,用于错误恢复调度的工具类FutureUtils | 适配器模式,akka |        |
| deployment       | 部署描述符和其工厂类,Task部署描述符包含输入口描述符集合和结果分区描述符集合,被用于ExecutionGraph |  |        |
| dispatcher       |                                                              |                                                              |        |
| entrypoint       |                                                              |                                                              |        |
| event       |                                                              |                                                              |        |
| execution       |                                                              |                                                              |        |
| executiongraph       |                                                              |                                                              |        |
| filecache       |                                                              |                                                              |        |
| heartbeat       |                                                              |                                                              |        |
| highavailability       |                                                              |                                                              |        |
| history       |                                                              |                                                              |        |
| instance       |                                                              |                                                              |        |
| io       |                                                              |                                                              |        |
| iterative       |                                                              |                                                              |        |
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

