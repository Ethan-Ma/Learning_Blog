# ClickHouse表引擎选取
## 概览
- ClickHouse提供了大约28种表引擎，各有各的用途，比如有Log系列用来做小表数据分析，MergeTree系列用来做大数据量分析，而Integration系列则多用于外表数据集成。再考虑复制表Replicated系列，分布式表Distributed等，纷繁复杂，新用户上手选择时常常感到迷惑。

## MergeTree系列
- MergeTree系列才是官方主推的存储引擎，支持几乎所有ClickHouse核心功能；
- MergeTree表引擎主要用于海量数据分析，支持数据分区、存储有序、主键索引、稀疏索引、数据TTL等；
- MergeTree支持所有ClickHouse SQL语法，但是有些功能与MySQL并不一致，比如在MergeTree中主键并不用于去重；
- 实例：
	- 如下建表DDL所示，test_tbl的主键为(id, create_time)，并且按照主键进行存储排序，按照create_time进行数据分区，数据保留最近一个月。
```
CREATE TABLE test_tbl (
  id UInt16,
  create_time Date,
  comment Nullable(String)
) ENGINE = MergeTree()
PARTITION BY create_time
ORDER BY  (id, create_time)
PRIMARY KEY (id, create_time)
TTL create_time + INTERVAL 1 MONTH
SETTINGS index_granularity=8192;
```
- 由于MergeTree采用类似LSM tree的结构，很多存储层处理逻辑直到Compaction期间才会发生。

### ReplacingMergeTree
- 为了解决MergeTree相同主键无法去重的问题，ClickHouse提供了ReplacingMergeTree引擎，用来做去重。
- 虽然ReplacingMergeTree提供了主键去重的能力，但是仍旧有以下限制：
	- 在没有彻底optimize之前，可能无法达到主键去重的效果，比如部分数据已经被去重，而另外一部分数据仍旧有主键重复；
	- 在分布式场景下，相同primary key的数据可能被sharding到不同节点上，不同shard间可能无法去重；
	- optimize是后台动作，无法预测具体执行时间点；
	- 手动执行optimize在海量数据场景下要消耗大量时间，无法满足业务即时查询的需求；
- 因此ReplacingMergeTree更多被用于确保数据最终被去重，而无法保证查询过程中主键不重复。

### CollapsingMergeTree
 
 ENGINE = CollapsingMergeTree(Sign)
- ClickHouse实现了CollapsingMergeTree来消除ReplacingMergeTree的限制。该引擎要求在建表语句中指定一个标记列Sign，后台Compaction时会将主键相同、Sign相反的行进行折叠，也即删除;
- CollapsingMergeTree将行按照Sign的值分为两类：Sign=1的行称之为状态行，Sign=-1的行称之为取消行;
- 每次需要新增状态时，写入一行状态行；需要删除状态时，则写入一行取消, 在后台Compaction时，状态行与取消行会自动做折叠（删除）处理。而尚未进行Compaction的数据，状态行与取消行同时存在;
- 因此为了能够达到主键折叠（删除）的目的，需要业务层进行适当改造：
	1) 执行删除操作需要写入取消行，而取消行中需要包含与原始状态行一样的数据（Sign列除外）。所以在应用层需要记录原始状态行的值，或者在执行删除操作前先查询数据库获取原始状态行；
	2) 由于后台Compaction时机无法预测，在发起查询时，状态行和取消行可能尚未被折叠；另外，ClickHouse无法保证primary key相同的行落在同一个节点上，不在同一节点上的数据无法折叠。因此在进行count(*)、sum(col)等聚合计算时，可能会存在数据冗余的情况。为了获得正确结果，业务层需要改写SQL，将count()、sum(col)分别改写为sum(Sign)、sum(col * Sign)。

### VersionedCollapsingMergeTree

 ENGINE = CollapsingMergeTree(Sign)
- 为了解决CollapsingMergeTree乱序写入情况下无法正常折叠问题，VersionedCollapsingMergeTree表引擎在建表语句中新增了一列Version，用于在乱序情况下记录状态行与取消行的对应关系。主键相同，且Version相同、Sign相反的行，在Compaction时会被删除。
- 与CollapsingMergeTree类似， 为了获得正确结果，业务层需要改写SQL，将count()、sum(col)分别改写为sum(Sign)、sum(col * Sign)。


### SummingMergeTree
- ClickHouse通过SummingMergeTree来支持对主键列进行预先聚合。在后台Compaction时，会将主键相同的多行进行sum求和，然后使用一行数据取而代之，从而大幅度降低存储空间占用，提升聚合计算性能。
- 值得注意的是：
	- ClickHouse只在后台Compaction时才会进行数据的预先聚合，而compaction的执行时机无法预测，所以可能存在部分数据已经被预先聚合、部分数据尚未被聚合的情况。因此，在执行聚合计算时，SQL中仍需要使用GROUP BY子句。
	- 在预先聚合时，ClickHouse会对主键列之外的其他所有列进行预聚合。如果这些列是可聚合的（比如数值类型），则直接sum；如果不可聚合（比如String类型），则随机选择一个值。
	- 通常建议将SummingMergeTree与MergeTree配合使用，使用MergeTree来存储具体明细，使用SummingMergeTree来存储预先聚合的结果加速查询。

### AggregatingMergeTree
- 与SummingMergeTree的区别在于：SummingMergeTree对非主键列进行sum聚合，而AggregatingMergeTree则可以指定各种聚合函数;
- AggregatingMergeTree的语法比较复杂，需要结合物化视图或ClickHouse的特殊数据类型AggregateFunction一起使用。在insert和select时，也有独特的写法和要求：写入时需要使用-State语法，查询时使用-Merge语法。

## Replicated 系列
- 对于 INSERT 和 ALTER 语句操作数据的会在压缩的情况下被复制；
- 而 CREATE，DROP，ATTACH，DETACH 和 RENAME 语句只会在单个服务器上执行，不会被复制：
	- The CREATE TABLE 在运行此语句的服务器上创建一个新的可复制表。如果此表已存在其他服务器上，则给该表添加新副本。
	- The DROP TABLE 删除运行此查询的服务器上的副本。
	- The RENAME 重命名一个副本。换句话说，可复制表不同的副本可以有不同的名称。


