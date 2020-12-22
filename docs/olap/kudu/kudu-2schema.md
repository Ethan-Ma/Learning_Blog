##背景
----
Kudu有着和MySQL等传统RDBMS类似的存储结构。表结构的设计对性能和稳定性起着决定性作用。
Kudu的表结构设计有三个重要概念：Column设计、PrimaryKey设计、Tablet设计。
其中 Column\PrimaryKey设计 跟传统数据库类似；Tablet设计 是分布式数据库特有的；
Tablet设计把数据分布在不同的机器上，对于不同的业务场景，不同的读写分布，其Tablet设计可能千差万别。
----

##Schema设计
比较好的Schema设计目标：
- 数据的随机读写均匀分布在不同Tablet上，充分利用分布式资源，使得吞吐量最大化；
- 每个Tablet负载相当；
- Scan操作读取最少的数据量，对于BI分析，还需要放弃写性能 来提升 Scan性能。

##Columns设计
- 表最大支持300列；
- 一旦创建，不能使用Alter table 修改列的类型和属性；
- 副本数在创建表时指定，后续无法改变；
- 表名、列名必须是有效的UTF-8字符串，最大支持256个字符；
- 创建表时要设置好每一列对应的数据类型，还可以针对每一列设置不同的压缩方式；
- 针对不同的类型，Kudu默认会选择不同的列编码方式，以达到最大的存储和检索效率；
| 列类型                 | 支持的编码方式                | 默认       |
| ---                    | ---                           | ---        |
| int8, int16, int32     | plain, bitshuffle, run length | bitshuffle |
| int64, unixtime_micros | plain, bitshuffle, run length | bitshuffle |
| float, double          | plain, bitshuffle             | bitshuffle |
| bool                   | plain, run length             | run length |
| string, binary         | plain, prefix, dictionary     | dictionary |

###Bitshuffle Encoding
该编码算法大概原理为按照数据出现频次来对比特重新分布，最终采用LZ4压缩落地。对于重复数据出现比较多的列，或者主键排序之后相邻单元差异较小，Bitshuffle是一个不错的选择。

###Run Length Encoding
该编码方式把相邻的相同元素按长度进行编码存储。比如原始数据为aaabbbbc，编码后变为a3b4c1。如某列出现的值类别很少，比如性别、国家等。用该编码方式较合适。

###Dictionary Encoding 
字典的编码方式把输入的字符串转换为[0-n)的整数，n为所有字符串去重后的个数。显而易见，n越小，字典的编码方式效率越高。另外当n比较大时，Kudu会自动的进行降级处理，编码方式自动降级为平铺的编码（Plain Encoding)

###Prefix Encoding
前缀编码，可以近似认为内部使用Trie（字典树）进行存储，当数据前缀相同部分较多时比较适合采用该编码方式。另外主键中的第一列是按前缀进行字典序排序，此时也可采用前缀编码。

##列压缩
- Kudu允许的落地压缩算法为LZ4、Snappy和zlib(gz)。默认情况下，Kudu不压缩数据。通常情况下压缩算法会提高空间利用率，但是会降低Scan性能。
- LZ4和Snappy比较类似，空间和时间有着很好地均衡，zlib有着较高的压缩比，但是Scan性能最差。
- 需要注意的是Bitshuffle Encoding已经在最后采用了LZ4，所以对于采用这种编码方式的列，无需再指定额外的压缩算法。

##PrimaryKey 设计
- 每一个Kudu的表都有且仅有一个主键，主键可以包含多个列，同时要求每一列的值都不能为空（non-nullable)，另外bool和浮点数也不能作为主键;
- 跟MySQL类似，主键具有唯一性，同一个主键只能对应一行数据，对于主键重复的数据insert会触发duplicate key errorl;
- Kudu不支持自增主键；
- 对于每个tablet的数据可以近似认为是按照主键有序存储的，主键字典序相近的数据会放在一起，利用这个特性可以极大提高Scan的性能；

##Tablet 设计
Kudu会按照 tablet规则将一个表切分成多个partition，优化一个表的 tablet规则需要考虑：随机读、随机写、Scan 三种操作，需要根据业务场景的不同做决定。

###随机写压力场景
对于随机写较多的场景，最重要的就是把写压力分担到不同的tablet中，通常采用 Hash Partitioning, Hash切片具有良好的随机性。

###随机读压力场景
对于随机读压力大的场景，通常情况读分布符合 齐夫定律，也就是28原则，80%的读集中在20%的数据上，则通常用Range Partitioning。

###小范围Scan场景
此类场景最好将同一Scan所需要的数据放置在同一tablet中，所以通常使用Hash partitioning；

###大范围Scan场景
此类场景最好是把数据分散到多个tablet中，能够充分利用分布式计算的能力；如果采用Hash方式Scan的压力会集中到一个tablet中；这种情况下最好采用先按Range Partitioning 再按Hash Partitioning 进行切片。

###多级切片
Range的切片通常与时间有关系；
Kudu支持多层的切片方式，Hash和Range可以结合使用，比如按照月份进行切片，然后对每个月的数据在按照Hash进行二级切片；

- 合理的切片可以让Scan操作跳过部分切片；
- 合理的切片可以防止写入分配到同一个tablet服务中。

###Tablet设计案例
CREATE TABLE metrics ( 
    host STRING NOT NULL, 
		metric STRING NOT NULL, 
		time INT64 NOT NULL, 
		value DOUBLE NOT NULL, 
		PRIMARY KEY (host, metric, time),
);
该表有四个字段，host、metric、time和value。主键包含三列(host、metric和time）

表1：采用hash切片，并且采用了line_id作为hash id（不指定hash id，主键即为hash id默认值）
该表共有50个切片。

create table speed1 (
	line_id string, 
	request_time timestamp, 
	idvisitor string, 
	primary key(line_id))
partition by hash partitions 50
stored as kudu;

表2:采用了range的切片方式，并且每天一个片。需要注意的是，因为把request_time加入了切片规则，所以主键之中必须包含request_time。

该表共有13个切片，分别对应13天的数据。

create table speed_test_2 (
	line_id string, 
	request_time timestamp, 
	idvisitor string, 
primary key(line_id,request_time) )
partition by range(request_time)(
	partition cast(1505130896 as timestamp) <= values>< cast(1505217296 as timestamp),
	partition cast(1505217296 as timestamp)><= values>< cast(1505303696 as timestamp),
	partition cast(1505303696 as timestamp)><= values>< cast(1505390096 as timestamp),
	partition cast(1505390096 as timestamp)><= values>< cast(1505476496 as timestamp),
	partition cast(1505476496 as timestamp)><= values>< cast(1505562896 as timestamp),
	partition cast(1505562896 as timestamp)><= values>< cast(1505649296 as timestamp),
	partition cast(1505649296 as timestamp)><= values>< cast(1505735696 as timestamp),
	partition cast(1505735696 as timestamp)><= values>< cast(1505822096 as timestamp),
	partition cast(1505822096 as timestamp)><= values>< cast(1505908496 as timestamp),
	partition cast(1505908496 as timestamp)><= values>< cast(1505994896 as timestamp),
	partition cast(1505994896 as timestamp)><= values>< cast(1506081296 as timestamp),
	partition cast(1506081296 as timestamp)><= values>< cast(1506167696 as timestamp),
	partition cast(1506167696 as timestamp)><= values>< cast(1506254096 as timestamp)
)stored as kudu;

表3
表3融合了表1和表2两种建表方式，切片方法既包含了hash，也包含了range。

该表共有13*3=39个切片，代表了13天的数据，每天3个hash切片。

create table speed4 ( 
	line_id string, 
	request_time timestamp, 
	idvisitor string, 
primary key(line_id,request_time))
partition by hash (line_id) partitions 3,range(request_time)(
	PARTITION cast(1505130896 as timestamp) <= values>< cast(1505217296 as timestamp),
	partition cast(1505217296 as timestamp)><= values>< cast(1505303696 as timestamp),
	partition cast(1505303696 as timestamp)><= values>< cast(1505390096 as timestamp),
	partition cast(1505390096 as timestamp)><= values>< cast(1505476496 as timestamp),
	partition cast(1505476496 as timestamp)><= values>< cast(1505562896 as timestamp),
	partition cast(1505562896 as timestamp)><= values>< cast(1505649296 as timestamp),
	partition cast(1505649296 as timestamp)><= values>< cast(1505735696 as timestamp),
	partition cast(1505735696 as timestamp)><= values>< cast(1505822096 as timestamp),
	partition cast(1505822096 as timestamp)><= values>< cast(1505908496 as timestamp),
	partition cast(1505908496 as timestamp)><= values>< cast(1505994896 as timestamp),
	partition cast(1505994896 as timestamp)><= values>< cast(1506081296 as timestamp),
	partition cast(1506081296 as timestamp)><= values>< cast(1506167696 as timestamp),
	partition cast(1506167696 as timestamp)><= values>< cast(1506254096 as timestamp)
)stored as kudu;
----
测试结论:
对于不同类型的查询sql，本次测试结果如下：
----
单次查询:
|请求类型|表1查询对应的切片数量|表2查询对应的切片数量|表3查询对应的切片数量|解释|
|---|---|---|---|---|
|select count(*) from table_name|50|13|39|表1、2、3的全部切片都有扫描|
|select * from table_name where line_id='xxx'|1|13|13|表一line_id是唯一主键，只需要查询一个切片，表2表3则不行。同时三张表line_id均是第一索引，所以查询操作都很快。|
|select * from table_name where idvisitor='xxx'|50|13|39|idvisitor不在主键之中，所以需要查询所有切片|
|select count(*) from table_name where request_time>adddate(now(), -5)|50|5|15|表1扫描了全部切片，表2和表3因为有range配置，所以只扫描了部分切片可以看到Kudu会根据where条件跳过部分分区。
对于带有时间where条件的大范围Scan查询而言，
可以看出，表2和表3是比较合适的。表3虽然扫描的切片比表2多，
但是扫描的总数据量是和表2一样的，同时表3能更好的利用多机资源，
可以把并发度从表2的5提高到15.|

----
通常情况下，应当尽量使用类似表3的结构来降低Scan操作所扫描的数据总量。

已知的限制:
- 列的数量最多300，越少越好
- 总切片数量不适宜太多，单个物理机最多承受1000个切片
- 每张表每片数据在1000w条左右较合适，根据列的数量，该建议需灵活配置
- 单个cell最大64KB
- 单行数据不能太大
- 描述用UTF-8编码之后不能超过256字节
- 主键对应的cell内容不可改变
- 主键包含那几列建表时需要设置好，后续不可改变
- 切片规则不可改变，但是Range切片可以增加切片或者删除切片
- 列的类型不能改变

