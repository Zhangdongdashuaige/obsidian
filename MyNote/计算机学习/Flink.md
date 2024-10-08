---
created: 2024-08-20
modified: 2024-09-14
tags:
  - Flink
official url: https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/
---

# Flink是什么
***
**Flink** 是 ***”数据流上的有状态计算“*** ，Apache Flink是一个 ***框架*** 和 ***分布式处理引擎*** ，用于对 ***无界*** 和 ***有界*** 数据流进行有状态计算。
![[Pasted image 20240830090512.png]]
#### 无界数据流：
1. 有定义流的开始，但没有定义流的结束；
2. 他们会无休止的产生数据；
3. 无界流的数据必须持续处理，即数据被摄取后需要立刻处理。我们不能等到所有数据都到达再处理，因为输入是无限的。
#### 有界数据流
1. 有定义流的开始，也有定义流的结束；
2. 有界流可以在摄取所有数据后再进行计算；
3. 有界流所有数据可以被排序，所以并不需要有序摄取；
4. 有界流处理通常被成为批处理。
#### 有状态流处理
把流处理需要的 ***额外数据保存成一个”状态”***，然后针对这条数据进行处理，并且 ***“更新状态”***。这就是所谓的 ***“有状态的流处理”***。
- 状态在内存中：优点，速度快；缺点：可靠性差。
- 状态在分布式系统中：有点。可靠性高；缺点，速度慢。
#### Flink特点
处理数据的目标是：***低延迟、高吞吐、结果的准确性和良好的容错性*** 。
- ***高吞吐和低延迟*** ：每秒处理数百万个事件，毫秒级延迟。
- ***结果的准确性*** ：Flink提供了 ***事件时间(event time)*** 和 ***处理时间(processing time)*** 语义。对于乱序事件流，时间时间语义仍然能提供一致且准确的结果。
- ***精确一次（exactly once）*** 的状态一致性保证。
- ***可以连接到最常用的外部系统*** ：如Kafka、Hive、JDBC、HDFS、Redis等。
- ***高可用*** ：本身高可用的设置，加上K8s，YARN和Mesos的紧密集成，在加上从故障中快速恢复和动态扩展任务的能力，Flink能做到以极少的停机时间7 X 24全天候运行。
#### Flink分层API
![[Pasted image 20240830092338.png]]
<br>
# Flink中的流类型以及相互转化
***
- ***DataStream：*** 
  Flink流处理api中最核心的数据结构。它代表一个运行在多个分区上的并行流。可以从 <font color="red"><b>StreamExecutionEnvironment</b></font> 通过 <font color="red"><b>env.addSource(SourceFunction)</b></font> 获得。
  DateStream上的转换操作都是逐条的，比如map(),flatmap(),filter()。DataStream也可以执行<font color="red"><b>rebalance</b></font> （再平衡，用于减轻数据倾斜）和 <font color="red"><b> broadcaseted </b></font> （广播）等分区转换。
- ***KeyedStream：***
  ***KeyedStream***用来表示根据指定的key进行分组的数据流。一个KeyedStream可以通过调用***DataStream.keyby()*** 来获得。而在KeyedStream上进行任何的transformation都将转变回DataStream。在实现中KeyedStream是把key的信息写入到了transformation中。每条记录只能访问所属key的状态，其上的聚合函数可以方便地操作和保存对应key的状态。
- ***WindowedStream & AllWindowedStream：***
  ***WindowedStream***代表了根据key分组，并且基于***WindowAssigner***切分窗口的数据流。所以***WindowedStream***都是从***KeyedStream***衍生而来的。而在***WindowedStream***上进行任何操作也都将转变回***DataStream***。
  Flink的窗口实现中会将到达的数据缓存在对应的窗口buffer中（一条数据可能对应多个窗口）。当到达窗口发送的条件时（由Trigger控制），Flink会对整个窗口中的数据进行处理。Flink在聚合类窗口有一定的优化，即不会保存窗口中的所有值，而是每到一个元素执行一次聚合函数，最终只保存一份数据即可。
  在key分组的流上进行窗口切分是比较常用的场景，也能够很好地并行化（不同的key上的窗口聚合可以分配到不同的task去处理）。不过有时候我们也需要在普通流上进行窗口的操作，这就是 ***AllWindowedStream***。***AllWindowedStream****是直接在DataStream上进行***windowAll(...)*** 操作。***AllWindowedStream*** 的实现是基于 ***WindowedStream*** 的（Flink 1.1.x 开始）。Flink 不推荐使用***AllWindowedStream***，因为在普通流上进行窗口操作，就势必需要将所有分区的流都汇集到单个的Task中，而这个单个的Task很显然就会成为整个Job的瓶颈
- ***JoinedStreams & CoGroupedStreams：***
  co-group侧重的是group，是对同一个key上的两组集合进行操作，而join侧重的是pair，是对同一个key上的每对元素进行操作。co-group比join更通用一些，因为join只是co-group的一个特例，所以join是可以基于co-group来实现的。提供join接口是因为用户更熟悉join，而且能够跟DataSetAPI保持一致，降低用户学习成本。
- ***ConnectedStreams：***
  <br>
# Flink部署
***
## Flink集群角色

#Flink Flink提交作业和执行任务，需要几个关键组件：
- <font color="red"><b> 客户端(Client) </b></font> ：代码由客户端 ***获取*** 并做转换，之后提交给JobManager
- <font color="red"><b> JobManager </b></font> ：对作业进行中央调度管理；而它获取到要执行的作业后，会进一步处理转换，然后分发任务给众多的Task Manager。
- <font color="red"><b> TaskManager </b></font>：实际工作角色
## Flink集群搭建

- 进入conf路径，修改flink-conf.yaml文件，指定JobManager节点
```
# JobManager节点地址
jobmanager.rpc.address: hadoop102
jobmanager.bind-host: 0.0.0.0
rest.address: hadoop102
rest.bind-address: 0.0.0.0

# TaskManager节点地址.需要配置为当前机器名
taskmanager.bind-host: 0.0.0.0
taskmanager.host: hadoop102
```
- 修改workers文件，指定TaskManager节点主机名
- 修改masters文件，指定master节点以及端口号
## Flink常用命令

```bash
# Flink集群客户端启动命令
bin/start-cluster.sh

# 使用命令行提交flink任务
bin/flink run -m linux1:port -c 主类名 jar包路径
```
## Flink部署模式

- ***会话模式（Session Mode）***
  启动一个集群，保持会话，在这个会话中通过客户端提交作业。集群启动时所有资源已经确定，在这个会话中提交的作业会竞争资源。
  会话模式比较适合于<font color="red"><b>单个规模小、执行时间短的大量作业</b></font>。
  ![[Session-mode.drawio.png]]
  
- ***单作业模式（Per-Job Mode）***
  会话模式因为资源共享会导致很多问题，为了更好的隔离资源，考虑为每一个job启动一个集群，这就是***单作业模式***。
  作业完成后，集群就会关闭，所有的资源也会释放。
  单作业模式的特性使得其在生产环境下运行更加稳定，所以是<font color="red"><b>实际场景的首选模式</b></font>。
  需要注意的是，Flink本身无法这样运行，所以单作业模式一般需要借助一些资源管理框架来启动集群，比如YARN、Kubernetes（K8S）。
  ![[Per-Jobmode.png]]
  
- ***应用模式（Application Mode）***
  ==前面提到的两种模式，应用代码都是在客户端上执行==。然后由客户端提交给JobManager的。但是这种方式==客户端需要占用大量的网络带宽==，去下载依赖和把二进制数据发送给JobManager；加上很多情况下我们提交作业用的是同一个客户端，==当作业较多时客户端所在的节点的资源消耗过多==。
  解决办法时，取代客户端，直接把应用提交到JobManager上运行。由JobManager解析代码。这就意味着我们需要为每一个提交的应用单独启动一个JobManager，也就是创建一个集群。这个JobManager只为执行这一个应用而存在，执行结束后JobManager也就关闭了，这就是所谓的应用模式。
  ![[Application Mode.png]]
``` bash
# 以会话模式启动Flink集群，启动集群后，可以在WebUI上提交作业
bin/start-cluster.sh 

# 命令行方式提交作业
bin/flink run -m hadoop102:8081 -c 主类名 jar包路径

###########################################################################
# 以应用模式启动Flink集群
bin/standalone-job.sh start --job-classname 主类名

# 启动TaskManager
bin/taskmanager.sh start

# 终止集群
bin/taskmanager.sh stop
bin/stanalone-job.sh stop
```
## YARN运行模式

YARN上部署的过程时：客户端把Flink应用提交给Yarn的ResourceManager，Yarn的ResourceManager会向Yarn的NodeManager申请容器。在这些容器上，Flink会部署JobManager和TaskManager的实例，从而启动集群。Flink会根据运行在JobManager上的作业所需要的Slot数量动态分配TaskManager资源。

**相关配置***
首先确认集群是否安装由Hadoop，保证Hadoop版本至少在2.2以上，并且集群中安装有HDFS服务。
1. 配置环境变量
``` bash
HADOOP_HOME=/opt/module/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop
export HADOOP_CALSSPATH=`hadoop classpath`
```
2. 启动Hadoop集群
``` bash
start-dfs.sh
start-yarn.sh
```

***会话模式部署***
YARN的会话模式部署时，首先需要申请一个YARN会话（YARN Session）来启动Flink集群。
1. 启动Hadoop集群
2. 执行脚本命令向YARN集群申请资源，开启一个YARN会话，启动Flink集群。
``` bash
# 启动一个YARN Session
bin/yarn-session.sh -nm tests

# 可用参数解读：
# -d：分离模式，Flink YARN客户端后台运行
# -jm(--jobManagerMemory)：配置Job Manager所需内存，默认单位MB
# -nm(--name)：配置在YARN UI界面上显示的任务名
# -qu(--queue)：指定YARN队列名
# -tm(--taskManager)：配置每个TaskManager所使用内存

# 通过命令行提交作业
bin/flink run -c 主类名 jar包
```   
 ==Flink1.11.0版本不再使用-n参数和-s参数分别指定TaskManager数量和slot数量，YARN会按照需求动态分配TaskManager和slot
 YARN Session启动之后会给出一个Web UI地址以及一个YARN application ID，如下所示，用户可以通过Web UI或者命令行两种方式提交作业==
 
***单作业模式部署***
在YARN环境中，由于有个外部平台做资源调度，所以我们也可以直接向YARN提交一个单独的作业，从而启动一个Flink集群。
``` bash
bin/flink run -d -t yarn-per-job -c 主类名 jar包

# 使用命令行查看作业
bin/flink list -t yarn-per-job -Dyarn.application.id=application_xxxx_yy

# 使用命令行取消作业
bin/flink cancel -t yarn-per-job -Dyarn.application.id=application_xxxx_yy <jobId>
```
==注意：如果启动过程中报如下异常==
```
Exception in thread “Thread-5” java.lang.IllegalStateException: Trying to access closed classloader. Please check if you store classloaders directly or indirectly in static fields. If the stacktrace suggests that the leak occurs in a third party library and cannot be fixed immediately, you can disable this check with the configuration ‘classloader.check-leaked-classloader’.

at org.apache.flink.runtime.execution.librarycache.FlinkUserCodeClassLoaders
```
==解决方法==
``` bash
# 在flink的/opt/module/flink-1.17.0/conf/flink-conf.yaml配置文件中设置
classloader.check-leaked-classloader：false
```

***应用模式部署***
应用模式与单作业模式类似，直接执行flink run-application命令即可。
``` bash
# 执行命令提交作业
bin/flink run-application -t yarn-application -c 主类名 jar包

# 查看作业
bin/flink list -t yarn-application -Dyarn.application.id=application_xxxx_yy

# 取消作业
bin/flink cancel -t yarn-application -Dyarn.application.id=application_xxxx_yy <jobId>

# jar包在HDFS上提交作业
bin/flink run-application -t yarn-application -Dyarn.provided.lib.dirs="hdfs://node:port/flink-dist" -c 主类名 hdfs路径
```

***历史服务器***
运行Flink job的集群一旦定制，只能去yarn或本地磁盘上产看日志，不再可以查看作业挂掉之前运行的Web UI，Flink提供了历史服务器用来在相应的Flink集群关闭后查询已完成作业的统计信息。我们知道只有当作业处于运行中的状态，才能查看到相关的Web UI统计信息。通过History Server我们才能查询这些已完成作业的统计信息，无论正常退出还是异常退出。
此外，它对外提供了REST API，它接受HTTP请求并使用JSON数据进行相应。Flink任务停止后，JobManager会将已经完成任务的统计信息进行存档，History Server进程则在任务停止后可以对任务统计信息进行查询。比如：最后一次的Checkpoint、任务运行时的相关配置。
1. 创建存储目录
``` bash
hadoop fs -mkdir -p /log/flink-job
```
2. 在flink-config.yaml中添加如下配置
``` bash
jobmanager.archive.fs.dir: hdfs://hadoop102:8020/logs/flink-job
historyserver.web.address: hadoop102
historyserver.web.port: 8082
historyserver.archive.fs.dir: hdfs://hadoop102:8020/logs/flink-job
historyserver.archive.fs.refresh-interval: 5000
```
3. 启动历史服务器
``` bash
bin/historyserver.sh start
```
4. 停止历史服务器
``` bash
bin/historyserver.sh stop
```


# Flink运行时架构
***
#以下信息全部来自于Flink官方文档

Flink 是一个分布式系统，需要有效分配和管理计算资源才能执行流应用程序。它集成了所有常见的集群资源管理器，例如[Hadoop YARN](https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html)，但也可以设置作为独立集群甚至库运行。

## Flink集群剖析

Flink运行时由两种类型的进程组成：一个JobManager和一个或多个TaskManager。
![[Pasted image 20240914103925.png]]
_Client_ 不是运行时和程序执行的一部分，而是用于准备数据流并将其发送给 JobManager。之后，客户端可以断开连接（_分离模式_），或保持连接来接收进程报告（_附加模式_）。客户端可以作为触发执行 Java/Scala 程序的一部分运行，也可以在命令行进程`./bin/flink run ...`中运行。

可以通过多种方式启动 JobManager 和 TaskManager：直接在机器上作为[standalone 集群](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/resource-providers/standalone/overview/)启动、在容器中启动、或者通过[YARN](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/resource-providers/yarn/)等资源框架管理并启动。TaskManager 连接到 JobManagers，宣布自己可用，并被分配工作。
#### JobManager
_JobManager_ 具有许多与协调 Flink 应用程序的分布式执行有关的职责：它决定何时调度下一个 task（或一组 task）、对完成的 task 或执行失败做出反应、协调 checkpoint、并且协调从失败中恢复等等。这个进程由三个不同的组件组成：
- **ResourceManager**
  _ResourceManager_ 负责 Flink 集群中的资源提供、回收、分配 - 它管理 **task slots**，这是 Flink 集群中资源调度的单位。Flink 为不同的环境和资源提供者（例如 YARN、Kubernetes 和 standalone 部署）实现了对应的 ResourceManager。在 standalone 设置中，ResourceManager 只能分配可用 TaskManager 的 slots，而不能自行启动新的 TaskManager。
- **Dispatcher**
  _Dispatcher_ 提供了一个 REST 接口，用来提交 Flink 应用程序执行，并为每个提交的作业启动一个新的 JobMaster。它还运行 Flink WebUI 用来提供作业执行信息。
- **JobMaster**
  _JobMaster_ 负责管理单个[JobGraph](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/concepts/glossary/#logical-graph)的执行。Flink 集群中可以同时运行多个作业，每个作业都有自己的 JobMaster。

始终至少有一个 JobManager。高可用（HA）设置中可能有多个 JobManager，其中一个始终是 _leader_，其他的则是 _standby_（请参考 [高可用（HA）](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/ha/overview/)）。

#### TaskManagers
_TaskManager_（也称为 _worker_）执行作业流的 task，并且缓存和交换数据流。
必须始终至少有一个 TaskManager。在 TaskManager 中资源调度的最小单位是 task _slot_。TaskManager 中 task slot 的数量表示并发处理 task 的数量。请注意一个 task slot 中可以执行多个算子。
## Tasks 和算子链

对于分布式执行，Flink 将算子的 subtasks _链接_成 _tasks_。每个 task 由一个线程执行。将算子链接成 task 是个有用的优化：它减少线程间切换、缓冲的开销，并且减少延迟的同时增加整体吞吐量。链行为是可以配置的；请参考[链文档](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/dev/datastream/operators/overview/#task-chaining-and-resource-groups)以获取详细信息。

下图中样例数据流用 5 个 subtask 执行，因此有 5 个并行线程。
![[Pasted image 20240914112430.png]]

## Task Slots和资源

每个 worker（TaskManager）都是一个 _JVM 进程_，可以在单独的线程中执行一个或多个 subtask。为了控制一个 TaskManager 中接受多少个 task，就有了所谓的 **task slots**（至少一个）。

每个 _task slot_ 代表 TaskManager 中资源的固定子集。例如，具有 3 个 slot 的 TaskManager，会将其托管内存 1/3 用于每个 slot。分配资源意味着 subtask 不会与其他作业的 subtask 竞争托管内存，而是具有一定数量的保留托管内存。注意此处没有 CPU 隔离；当前 slot 仅分离 task 的托管内存。

通过调整 task slot 的数量，用户可以定义 subtask 如何互相隔离。每个 TaskManager 有一个 slot，这意味着每个 task 组都在单独的 JVM 中运行（例如，可以在单独的容器中启动）。具有多个 slot 意味着更多 subtask 共享同一 JVM。同一 JVM 中的 task 共享 TCP 连接（通过多路复用）和心跳信息。它们还可以共享数据集和数据结构，从而减少了每个 task 的开销。
![[Pasted image 20240914135423.png]]
默认情况下，Flink 允许 subtask 共享 slot，**即便它们是不同的 task (即算子)的 subtask（子任务）**，只要是来自于同一作业即可（==即同一个作业的不同算子的子任务可以共享一个slot==）。结果就是一个 slot 可以持有整个作业管道。允许***slot 共享***有两个主要优点：

- Flink 集群所需的 task slot 和作业中使用的最大并行度恰好一样。无需计算程序总共包含多少个 task（具有不同并行度）。

- 容易获得更好的资源利用。如果没有 slot 共享，非密集 subtask（_source/map()_）将阻塞和密集型 subtask（_window_） 一样多的资源。通过 slot 共享，我们示例中的基本并行度从 2 增加到 6，可以充分利用分配的资源，同时确保繁重的 subtask 在 TaskManager 之间公平分配。
![[Pasted image 20240914135451.png]]
## Flink 应用程序执行

_Flink 应用程序_ 是从其 `main()` 方法产生的一个或多个 Flink 作业的任何用户程序。这些作业的执行可以在本地 JVM（`LocalEnvironment`）中进行，或具有多台机器的集群的远程设置（`RemoteEnvironment`）中进行。对于每个程序，`ExecutionEnvironment` 提供了一些方法来控制作业执行（例如设置并行度）并与外界交互（请参考 [Flink 程序剖析](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/dev/datastream/overview/#anatomy-of-a-flink-program) ）。

Flink 应用程序的作业可以被提交到长期运行的 [Flink Session 集群](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/concepts/glossary/#flink-session-cluster)、专用的 [Flink Job 集群](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/concepts/glossary/#flink-job-cluster) 或 [Flink Application 集群](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/concepts/glossary/#flink-application-cluster)。这些选项之间的差异主要与集群的生命周期和资源隔离保证有关。
#### Flink Session 集群
- **集群生命周期**：在 Flink Session 集群中，客户端连接到一个预先存在的、长期运行的集群，该集群可以接受多个作业提交。即使所有作业完成后，集群（和 JobManager）仍将继续运行直到手动停止 session 为止。因此，Flink Session 集群的寿命不受任何 Flink 作业寿命的约束。
- **资源隔离**：TaskManager slot 由 ResourceManager 在提交作业时分配，并在作业完成时释放。由于所有作业都共享同一集群，因此在集群资源方面存在一些竞争 — 例如提交工作阶段的网络带宽。此共享设置的局限性在于，如果 TaskManager 崩溃，则在此 TaskManager 上运行 task 的所有作业都将失败；类似的，如果 JobManager 上发生一些致命错误，它将影响集群中正在运行的所有作业。
- **其他注意事项**：拥有一个预先存在的集群可以节省大量时间申请资源和启动 TaskManager。有种场景很重要，作业执行时间短并且启动时间长会对端到端的用户体验产生负面的影响 — 就像对简短查询的交互式分析一样，希望作业可以使用现有资源快速执行计算。

> 以前，Flink Session 集群也被称为 ___session 模式___ 下的 Flink 集群。
#### Flink Job 集群
- **集群生命周期**：在 Flink Job 集群中，可用的集群管理器（例如 YARN）用于为每个提交的作业启动一个集群，并且该集群仅可用于该作业。在这里，客户端首先从集群管理器请求资源启动 JobManager，然后将作业提交给在这个进程中运行的 Dispatcher。然后根据作业的资源请求惰性的分配 TaskManager。一旦作业完成，Flink Job 集群将被拆除。
- **资源隔离**：JobManager 中的致命错误仅影响在 Flink Job 集群中运行的一个作业。
- **其他注意事项**：由于 ResourceManager 必须应用并等待外部资源管理组件来启动 TaskManager 进程和分配资源，因此 Flink Job 集群更适合长期运行、具有高稳定性要求且对较长的启动时间不敏感的大型作业。

> 以前，Flink Job 集群也被称为 _job (or per-job) 模式_下的 Flink 集群。

> Kubernetes 不支持 Flink Job 集群。 请参考 [Standalone Kubernetes](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/resource-providers/standalone/kubernetes/#per-job-cluster-mode) 和 [Native Kubernetes](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/resource-providers/native_kubernetes/#per-job-cluster-mode)。

#### Flink Application 集群 
- **集群生命周期**：Flink Application 集群是专用的 Flink 集群，仅从 Flink 应用程序执行作业，并且 `main()`方法在集群上而不是客户端上运行。提交作业是一个单步骤过程：无需先启动 Flink 集群，然后将作业提交到现有的 session 集群；相反，将应用程序逻辑和依赖打包成一个可执行的作业 JAR 中，并且集群入口（`ApplicationClusterEntryPoint`）负责调用 `main()`方法来提取 JobGraph。例如，这允许你像在 Kubernetes 上部署任何其他应用程序一样部署 Flink 应用程序。因此，Flink Application 集群的寿命与 Flink 应用程序的寿命有关。
- **资源隔离**：在 Flink Application 集群中，ResourceManager 和 Dispatcher 作用于单个的 Flink 应用程序，相比于 Flink Session 集群，它提供了更好的隔离。

> Flink Job 集群可以看做是 Flink Application 集群”客户端运行“的替代方案。

## 作业提交流程

#### Standalone会话模式作业提交流程
![[Pasted image 20241008160026.png]]
#### 逻辑流图/作业图/执行图/物理流图
1. 逻辑流图
   根据用户通过DataStream API编写的代码生成的最初的DAG图，用来表示程序的拓扑结构。这一步一般在客户端完成。
2. 作业图
   StreamGraph经过优化后生成的就是作业无（JobGraph），这是提交给JobManager的数据结构，确定了当前作业中所有任务的划分。主要的优化为：将多个符合条件的节点链接在一起合并成一个任务节点，形成算子链，这样可以减少数据交换的消耗。JobGraph一般也是在客户端生成的，在作业提交时传递给JobMaster。
3. 执行图
   JobMaster收到JobGraph后，会根据它生成执行图（ExecutionGraph）。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。与JobGraph最大的区别就是按照并行度对并行子任务进行了拆分，并明确了任务间数据传输的方式。
4. 物理图
   JobMaster生成执行图后，会将它分发给TaskManager；各个TaskManager会根据执行图部署任务，最终的物理执行过程也会形成一张“图”，一般就叫作物理图（Physical Graph）。这只是具体执行层面的图，并不是一个具体的数据结构。
   物理图主要就是在执行图的基础上，进一步确定数据存放的位置和收发的具体方式。有了物理图，TaskManager就可以对传递来的数据进行处理计算了。

## 基于时间的合流-双流连结（Join）

在interval-join时，只要数据的**事务时间**小于当前的**watermark**，则该条数据为迟到数据，不对其进行处理，可以在测输出流中将其输出。

# 处理函数
***
# 状态
***
## 什么是状态

Flink中算子可以分为 ***有状态*** 和 ***无状态*** 两种。基本转换算子计算时不依赖其他数据，属于无状态算子；而有状态的算子，除了当前数据之外，还需要一些其他数据来得到计算结果，这里的其他数据，就是 ***状态*** 。

有状态算子的一般处理流程，具体步骤如下：

1. 算子任务接收到上游发来的数据；
2. 获取当前状态；
3. 根据业务逻辑进行计算，更新状态；
4. 得到计算结果，输出发送到下游任务

## 状态的分类

## 托管状态（Managed State）和原始状态（Raw State）

## 状态后端（HashMapStateBackend/RocksDB）

(1)  

(2)




# 容错机制
***
## 检查点（Checkpoint）

[Flink中Barrier对齐机制_flink barrier 对齐-CSDN博客](https://blog.csdn.net/qq_42009405/article/details/122850469)

Barrier对齐的精准一次
Barrier不对齐的至少一次
Barrier不对齐的至少一次


## 状态一致性

**一致性的概念和级别**
一致性其实就是结果的正确性，一般从数据丢失、数据重复来评估。
一般来说，状态一致性有三种级别
1. 最多一次（At-Most-Once）
2. 至少一次（At-Least-Once）
3. 精确一次（Exactly-Once）




# FlinkSQL
***

