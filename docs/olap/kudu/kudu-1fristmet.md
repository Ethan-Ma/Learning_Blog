## 背景

###大数据两种存储方式：
1. 静态数据：以HDFS引擎作为存储引擎，适用于高吞吐量的离线大数据分析场景，这类存储的局限是数据无法进行随机的读写；
2. 动态数据：以HBase、Cassandra 作为存储引擎，适用于大数据随机读写场景，这类存储的局限是批量读取吞吐量远不如HDFS，不适用于批量数据分析的场景。
------
> HBase 低延迟随机读写，但不适用于给予SQL的数据分析；
> HDFS 高吞吐连续读写，但是列式存储不能更新数据，随机读写性能低；
> HBase 实时读写 HDFS 数据分析
-------
真实场景中，既需要随机读写，又需要批量分析的大数据场景，一个常见的方案是：
数据实时写入HBase，实时的数据更新也在HBase完成，为了应对OLAP需求，定时（通常是T+1或者T+H）将HBase数据写成静态的文件（如：Parquet）导入到OLAP引擎（如：HDFS）。
> 架构复杂：成本高、空间浪费、数据安全挑战
> 时效性差
> 难以应对后续更新
--------

## 介绍
Kudu不但提供了行级的插入、更新、删除API，同时也提供了接近parquet性能的批量扫描操作，
既可以随机读写，又可以满足数据分析的需求。定位是 Fast Analytics on fast data.
1. 它是一种 存储结构化数据 的存储系统；
2. 可以定义主键（一个表必须指定至少一个字段作为主键），但是不支持二级索引；
3. 表中每个字段都是强类型的，而不是类似于NoSQL的"everything is byte"。这可以带来两点好处：
>> 确定的列类型可以进行类型特有的编码，节省空间；
>> 可以提供SQL-like 元数据给上层查询工具，比如BI工具；
4. 可以使用Insert、Update、Delete API进行写操作，但都必须指定主键；批量的删除和更新操作需要依赖更高层次的组件（比如Impala、Spark）
5. Kudu支持但航事务，不支持多行事务（Kudu中对多行操作不满足ACID原则中的原子性），也不支持事务回滚，这点与HBase相同；
6. 一致性模型：默认是snapshot consistency 和 external consisitancy.
> Snapshot Consistency 快照一致性只能保持单个Client可以看到最新数据，不能保证多个Client每次取出的数据都是最新的；
> External Consistency 提供了两种方式：
>> 1. 在client 之间传递timestamp token，复杂度高； 
>> 2. 通过commit-wait方式，类似于Google的Spanner，解决思路是把不同机器的误差时间控制在一个确定的很小的范围内。
------
> 控制时间误差的方案为TrueTime，它通过硬件（GPS和原子钟）和软件的结合，保证获取到的时间在较小误差（±4ms）内绝对正确.
![truetime](https://gitee.com/ethanjh/pictures/raw/master/truetime.jpg)
> 如上图所示，在一个事务开始获取锁之后，生成事务的时间版本是 t=TT.now().latest,然后开始执行事务的具体操作，但是一个事务的结束并不只是由事务本身的时间消耗决定，它还要保证后续的事务时间版本不会早于自己，因此，事务需要等待直到TT.now().earliest>s后，才算真正结束。根据整个commit-wai过程可以知道，整个事务提交过程需要等待2倍的平均误差时间（ε），TrueTime的平均误差时间是4ms，因此一次commit-wait需要至少8ms。而且TrueTime需要昂贵的硬件设备支持，目前Kudu通过纯软件算法的方法来实现时钟算法，为HybridTime，但这个时间误差较大，实际应用场景的限制就多了。
--------

##架构
Kudu中存在两个角色：
> Master Server: 负责集群管理、元数据管理等
> Tablet Server：负责数据存储，并提供数据读写服务

为了实现分区容错性，跟其他大数据产品一样，对于每个角色，都设置特定数量的副本。各个副本通过Raft协议来保证数据的一致性。Raft协议与ZAB类似，都是Paxos协议的工程简化版本。

Kudu Client 与服务端交互，先从Master Server获取元数据信息，然后去Tablet Server 读写数据：
![Kudu_client](https://gitee.com/ethanjh/pictures/raw/master/kudu_cli_%E4%BA%A4%E4%BA%92.jpg)

##数据分区策略
一般的数据分区策略分为两种:
> Range Partitioning: 按照字段值范围进行分区，HBase就采用这种方式；
>> 优势在于批量读时可以把大部分的读变成对同一个tablet的顺序读，能够提升数据读取的吞吐量；而且按照范围分区，可以很方便的进行分区扩展；
>> 缺点是同一个范围内的数据写入都会落在单个tablet上，写的压力大，速度慢。
> Hash Partitioning: 按照字段的Hash值进行分区，Cassandra就是采用这种方式；
>> 优势是 数据的写入会被均匀的分散到各个tablet上，写入速度快；
>> 缺点是 对于顺序读的场景就不太适用，因为数据分散，一次顺序读需要将各个tablet中的数据分别读取并合并，吞吐量低；并且Hash分区无法应对分区扩展的情况。
![partitioning](https://gitee.com/ethanjh/pictures/raw/master/partitioning.jpg)

Kudu 支持用户对一个表 指定一个 范围分区规则 和多个 Hash分区规则，如图：
![multi_partitioning](https://gitee.com/ethanjh/pictures/raw/master/multi_partitioning.jpg)






















