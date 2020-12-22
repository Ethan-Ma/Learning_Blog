##核心组件

###Driver
Driver是一个线程，在这个线程中反射执行了创建上下文的main方法；
Driver在Spark作业执行中主要负责：
1. 将用户程序转化为作业(job);
2. 在Executor之间调度任务(task);
3. 跟踪Executor执行情况;
4. 通过UI展示查询运行情况。

###Executor
Executor其实是一个JVM进程 org.apache.spark.executor.CoarseGrainedExecutorBackend,
负责在Spark中运行具体任务。Task是发给Executor对应的进程，进程进而启动一个线程池，每个线程执行每个task。线程池共享进程内存，所以可以用来广播变量。
Executor有两个核心功能：
1. 负责运行 组成Spark应用的任务，并将结果返回给Driver进程；
2. 通过自身的块管理器(Block Manager)为用户程序缓存RDD。RDD是直接缓存在Executor进程内存中的，因此Task可以在运行时利用缓存数据加速计算。

##Spark通用运行流程
![spark运行流程1]()
![spark运行流程2]()

1. 不论Spark以何种模式进行部署，任务提交后，都会先启动Driver进程；
2. 随后Driver进程向 集群管理器 注册应用程序，之后集群管理器 根据任务的配置文件分配Executor并启动(启动的是Executor Backend 后台)；
3. 当Driver所需要的资源全部满足后，Driver开始执行main函数，Spark查询为 懒执行，当执行到 action 算子时开始反向推算，根据宽依赖进行Stage的划分，随后每一个Stage对应一个taskSet，taskSet中有多个task，根据本地化原则，task会被分发到 指定的Executor去执行，在任务执行中，Executor也会不断与Driver进行通信，报告任务运行情况。

##Spark运行模式配置
| Master Url           | Meaning                                                                                                                                                                      |
| :---                 | :----                                                                                                                                                                        |
| local                | 本地运行，只有一个工作进程，无并行计算能力；                                                                                                                                 |
| local[K] 或 local[*] | 本地运行，K个工作进程，通常K为CPU的核数；                                                                                                                                    |
| yarn-client          | 在Yarn集群上运行，Driver进程在本地，Executor进程在Yarn集群上，部署模式必须使用固定值：--deploy-mode client。Yarn集群地址必须在HADOOP_CONF_DIR 或 YARN_CONF_DIR变量里定义； |
| yarn-cluster         | 在Yarn集群上运行，Driver进程和Executor进程都在Yarn上，部署模式：--deploy-mode cluster。Yarn集群地址必须在HADOOP_CONF_DIR 或 YARN_CONF_DIR变量里定义。                        |

##Yarn模式运行机制
###Yarn Client模式
![yarn_cluster]()
![yarn_client]()
Application Master 是Yarn中的核心概念，任何要在Yarn上启动的作业类型（包括MR和Spark）都必须有一个Application Master。每种计算框架（MR或Spark）如果想要在Yarn上执行自己的计算应用，那么就必须自己实现和提供一个Application Master，相当于是自己实现了Yarn提供的接口，Spark自己开发的一个类。
Spark在Yarn-client模式下。Application 的注册(Executor的申请)和计算task的调度，是分开的；standalone模式下，这两个操作都是Driver负责的；ApplicationMaster(Executor  launcher)负责Executor的申请，Driver负责Job和Stage的划分以及Task的创建、分配和调度。

##Spark通讯架构
使用Netty通讯框架作为内部通讯组件，Spark基于Netty新型的RPC框架 借鉴了Akka中的设计，是基于Actor模型的。
----
BIO: 阻塞式IO
NIO: 非阻塞式IO
AIO: 异步非阻塞式IO，Netty使用的方式。
----
![RPC]()






