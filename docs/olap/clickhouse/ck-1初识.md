# ClickHouse

## 1. 核心特性
### Tips
1. 在索引方面，使用了LSM树所使用到的稀疏索引；
2. 在数据文件的设计上，则沿用了LSM树中的数据段的思想，即数据段内数据有序，借助稀疏索引定位数据段；
3. 基于上述基础，OLAP Server引入列式存储，将索引文件和数据文件按照列字段的粒度进行分拆，减少数据读取的范围；

### 1.1 CK不适用场景
1. 不支持事务；
2. 不擅长根据主键按行进行查询（支持），所以不应该把CK当作Key-Value数据库使用；
3. 不擅长按行删除数据（支持）。

### 1.2 完备的DBMS功能
CK拥有完备的管理功能，称得上是一个DBMS，而不仅仅是一个数据库：
1. DDL（数据定义语言）：可以动态地创建、修改或删除数据库、表和视图，无需重启服务；
2. DML（数据操作语言）：可以动态地查询、插入、修改或删除数据；
3. 权限管理：可以按照用户设置数据库或表的权限；
4. 数据备份和恢复：提供了数据备份导出与导入恢复机制，满足生产环境需求；
5. 分布式管理：提供集群模式，能够自动管理多个数据节点;
6. CK采用关系模型和SQL语言。

### 1.3 列式存储与数据压缩
1. 他俩是相伴而生的，一般来说 列式存储是数据压缩的前提；
2. 列式存储可以有效地减少查询时所扫描的数据量；
3. 属于同一字段的数据拥有相同的数据类型和实现语义，重复项的可能性更高，则可压缩率就越高，数据体量就越小，则数据在网络中传输越快，对网络带宽和磁盘IO的压力就越小；
4. CK采用列式存储，列与列之间由不同文件分别保存（主要是指MergeTree表引擎），数据默认使用[LZ4算法](./docs/olap/clickhouse/LZ4.md)压缩，列式存储除了降低IO和存储压力之外，还为向量化执行做好了铺垫； 

### 1.4 向量化执行引擎
1. 向量化执行，可以简单地看作一项消除程序中循环的优化。简单例子：非向量化执行是用1台机器循环制作n次，向量化执行是用n台机器执行1次。
2. 向量化执行需要用到CPU的SIMD指令，Single Instruction Multiple Data,即单条指令操作多条数据，它的原理是在CPU寄存器层面实现数据的并行操作。
（CK目前使用SSE4.2指令集实现向量化执行）

### 1.5 数据分片
1. CK在数据存储方面，既支持分区（纵向扩展，利用多线程），也支持分片（横向扩展，利用分布式原理）；
2. CK支持分片，而分片则依赖集群，每个分片对应一个服务节点，分片数量上限取决于节点数量（1个分片只能对应一个服务节点）；
3. CK没有高度自动化的分片功能，CK提供了本地表(Local Table)和分布式表(Distributed Table)的概念，一张本地表等同于一份数据的分片；而分布式表本身不存储数据，它是本地表的访问代理，作用类似于分库中间件；借助分布式表，能够代理访问多个数据分片，从而实现分布式查询；（业务初期数据量小可以使用单个节点的本地表，数据量增长再通过新增数据分片分流数据，并通过分布式表实现分布式查询）；
4. CK采用Multi-Master多主架构，客户端访问任何一个节点都能得到相同效果，天然规避了单点故障问题；

## 2. 架构设计
### 2.1 Column 与 Field 与 DataType
1. 内存中的一列数据由一个Column对象表示，Column对象分为接口和实现两部分；
2. 如果要操作单个具体数值(也就是单列中的一行数据)，则需要Field对象，它代表一个单值；与Column对象的泛化设计思路不同，Field对象是用了聚合的设计模式;
3. DataType负责数据的序列化与反序列化,但是它不直接负责数据的读取，而是从Column或Field对象获取。

### 2.2 Block与Block流
1. Column 和 Field组成了数据的基本映射单元，但还缺少数据类型、列名等信息；Block对象可以看作是数据表的子集，本质是由 数据对象、数据类型 和列名 组成的三元组(即 Column、DataType 和 列名字符串)；
2. Column提供了数据的读取能力，DataType提供了数据正反序列化，通过Block对象就可以完成一系列的数据操作；
3. 有了Block这一层对象封装之后，Block流就水到渠成，流操作有两组顶层接口：IBlockInputStream 负责数据的读取和关系运算，IBlockOutputStream负责将数据输出的下一个环节；

### 2.3 Table与Parser与Interpreter
1. 底层设计中并没有Table对象，它直接使用 IStorage接口指代数据表；在数据查询时，IStorage负责根据AST查询语句的指示要求返回指定的原始数据，后续对数据的进一步加工、计算和过滤，则会统一交给Interpreter解释器对象处理；
2. Parser分析器负责创建AST对象，而Interpreter解释器负责解释AST，并进一步创建查询的执行管道；

### 2.4 Cluster与Replication
1. CK集群由分片(Shard)组成，而每个分片又通过副本(Replica)组成；
2. CK的一个节点只能拥有一个分片，也就是说要实现1个分片、1个副本，则至少需要部署2个服务节点；
3. 分片只是一个逻辑概念，其物理承载还是由副本承担的；

## 2.5 CK为何快
1. CK会在内存中进行Group BY，并且使用HashTable装载数据；
2. 以字符串为例，对于常量，使用Volnitsky算法，对于非常量，使用CPU的向量化执行SIMD，暴力优化；正则匹配使用re2和hyperscan算法，性能是算法选择的首要目标；
3. 列式存储、向量化执行引擎 和 表引擎 都是它的杀手锏。

## 3. 数据定义
### 3.1 数据库引擎

数据库目前支持5中引擎：
1. Ordinary： 默认引擎，在此数据库下可以使用任意类型的表引擎；
2. Dictionary: 字典引擎，此类数据库会自动为所有数据字典创建他们的数据表；
3. Memory：内存引擎，用于存放临时数据，此类数据库下的数据表只会停留在内存中，不会涉及任何磁盘操作，当服务重启后数据会被清除；
4. Lazy： 日志引擎，此类数据库只能使用Log系列的表引擎；
5. MySQL：此类数据库会自动拉取远端MySQL中的数据，并为它们创建MySQL表引擎的数据表。

## 4. 数据字典

数据字典是以键值和属性映射的形式定义数据；字典中数据会主动或被动加载到内存（数据是在CK启动时加载还是在首次查询时加载 由参数设置决定），并支持动态更新；
由于字典常驻内存的特性，所以它适合保存常量或经常使用的维度数据，以避免不必要的JOIN查询。
### 4.1 内置字典

内置字典默认关闭，可以通过将config.xml文件中path_to_regions_hierarchy_file 和 path_to_regions_names_files两个配置打开来开启，这两项配置是惰性加载的，只有当字典首次被查询时才会出发加载操作。

### 4.2 外部扩展字典

1. 外部扩展字典是以插件形式注册到CK中，由用户自定义数据模式及数据来源。
2. 扩展字典由config.xml文件中的dictionaries_config配置来指定，默认情况下CK会自动识别并加载/etc/clickhouse-server目录下所有以_dictionary.xml结尾的配置文件。同时CK也能够动态感知此目录下配置文件的各种变化，并支持不停机在线更新配置文件。
3. 在单个字典配置文件内可以定义多个字典，其中每一个字典由一组dictionary元素定义，dictionary元素中包含5个子元素，均为必填。
4. 扩展字典目前不支持增量更新，但部分数据源能够依照标示判断，只有在数据源发生实质变化后才实施更新，这个判断源数据是否被修改的标识在字典内部称为previous。
（对于文件数据源，previous的值来自系统文件的修改时间；对于MySQL(InnoDB)等数据源，previous的值源于invalidate_query中定义的SQL语句。）

### 4.3 扩展字典的基本操作

1. 源数据查询：通过system.dictionaries系统表，可以查询扩展字典的元数据信息；
2. 数据查询：通过字典函数获取字典数据；
3. 字典表：字典表是使用Dictionary表引擎的数据表，通过这张表能查询到字典中的数据；
4. 使用DDL查询创建字典：19.17.4.11之后CK支持使用DDL查询创建字典。

## 5. MergeTree原理解析

表引擎决定了一张数据表的最终“性格”，比如数据表拥有何种特性、数据以何种形式被存储以及如何被加载；
在众多表引擎中，又属合并树(MergeTree)表引擎及其家族系列(*MergeTree)最为强大，在生产环境中大部分会用此系列的表引擎；
MergeTree作为合并树家族中最基础的表引擎，提供了主键索引、数据分区、数据副本和数据采样等基本能力；
### 5.1 MergeTree创建方式与存储结构
MergeTree在写入数据时，总会以数据片段写入磁盘，且数据片段不可修改，为了避免片段过多，CK后台会定期合并数据片段，属于相同分区的数据片段会被合并成一个新片段；
#### 5.1.1 MergeTree创建方式
表的创建方式大致相同，但需要ENGINE=MergeTree(), MergeTree 引擎没有参数;  
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]  
(  
    &nbsp; &nbsp; &nbsp; name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],  
    &nbsp; &nbsp; &nbsp; &nbsp; name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],  
    &nbsp; &nbsp; &nbsp; &nbsp; ...  
    &nbsp; &nbsp; &nbsp; &nbsp; INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,  
    &nbsp; &nbsp; &nbsp; &nbsp; INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2  
) ENGINE = MergeTree()  
ORDER BY expr  
[PARTITION BY expr]  
[PRIMARY KEY expr]  
[SAMPLE BY expr]  
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]  
[SETTINGS name=value, ...]  
-----
1. PARTITION BY [选填]：分区键，支持单列、多个列的元组以及列表达式；如果不声明分区键，则CK会生成一个名为all的分区；合理使用数据分区，可以有效减少查询时数据文件的扫描范围；
2. ORDER BY [必填]：排序键，支持单列、多个列的元组以及列表达式；默认情况下，主键(PRIMARY KEY)与排序键相同；
3. PRIMARY KEY[选填]：主键，声明之后会依照主键字段生成一级索引，用于加速表查询；默认情况下，主键(PRIMARY KEY)与排序键相同，所以通常直接使用ORDER BY 代为指定主键，无须刻意声明；与其他数据库不同，MergeeTree主键允许存在重复数据(ReplacingMergeTree可以去重)；
4. SAMPLE BY [选填]：抽样表达式，用于声明数据以何种标准进行采样，如果用了此配置，那么在主键的配置中也需要声明同样的表达式；
5. SETTINGS — 控制 MergeTree 行为的额外参数：  
index_granularity — 索引粒度。索引中相邻的『标记』间的数据行数。默认值，8192 。参考数据存储。  
index_granularity_bytes — 索引粒度，以字节为单位，默认值: 10Mb。如果想要仅按数据行数限制索引粒度, 请设置为0(不建议)。  
enable_mixed_granularity_parts — 是否启用通过 index_granularity_bytes 控制索引粒度的大小。在19.11版本之前, 只有 index_granularity 配置能够用于限制索引粒度的大小。当从具有很大的行（几十上百兆字节）的表中查询数据时候，index_granularity_bytes 配置能够提升ClickHouse的性能。如果你的表里有很大的行，可以开启这项配置来提升SELECT 查询的性能。  
use_minimalistic_part_header_in_zookeeper — 是否在 ZooKeeper 中启用最小的数据片段头 。如果设置了 use_minimalistic_part_header_in_zookeeper=1 ，ZooKeeper 会存储更少的数据。更多信息参考『服务配置参数』这章中的 设置描述 。  
min_merge_bytes_to_use_direct_io — 使用直接 I/O 来操作磁盘的合并操作时要求的最小数据量。合并数据片段时，ClickHouse 会计算要被合并的所有数据的总存储空间。如果大小超过了 min_merge_bytes_to_use_direct_io 设置的字节数，则 ClickHouse 将使用直接 I/O 接口（O_DIRECT 选项）对磁盘读写。如果设置 min_merge_bytes_to_use_direct_io = 0 ，则会禁用直接 I/O。默认值：10 * 1024 * 1024 * 1024 字节。  
merge_with_ttl_timeout — TTL合并频率的最小间隔时间，单位：秒。默认值: 86400 (1 天)。  
write_final_mark — 是否启用在数据片段尾部写入最终索引标记。默认值: 1（不建议更改）。  
merge_max_block_size — 在块中进行合并操作时的最大行数限制。默认值：8192  
storage_policy — 存储策略。 参见 使用具有多个块的设备进行数据存储.  
min_bytes_for_wide_part,min_rows_for_wide_part 在数据片段中可以使用Wide格式进行存储的最小字节数/行数。你可以不设置、只设置一个，或全都设置。参考：数据存储  


### 5.2 MergeTree的存储结构

MergeTree表引擎中的数据会按照分区目录的形式保存到磁盘，完整存储结构如下：
--------
table_name<br>
|<br>
|-partition_1<br>
| |-checksums.txt&nbsp; \ <br> 
| |-columns.txt&nbsp; &nbsp; &nbsp; &nbsp; | <br> 
| |-count.txt&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; | <br> 
| |-primary.idx&nbsp; &nbsp; &nbsp; &nbsp; |  <br>
| |-[Column].bin&nbsp; &nbsp; |&nbsp; 基础文件<br>
| |-[Column].mrk&nbsp; &nbsp; &nbsp; |  <br>
| |-[Column].mrk2&nbsp; /  <br>
| |  <br>
| |-partition.dat&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; \ 使用分区键时才会生成 <br> 
| |-minmax_[Column].idx&nbsp; &nbsp; /  <br>
| |  <br>
| |-skp_idx_[Column].idx&nbsp; &nbsp; &nbsp; \	使用二级索引时才会生成  <br>
| |-skp_idx_[Column].mrk&nbsp; &nbsp; /  <br>
|  <br>
|-partition_2  <br>
|  <br>
|-partition_n  <br>
--------
上图可以看出，一张表的物理结构分为3层：数据表目录、分区目录 和 分区下具体的数据文件:
1. partition：分区目；
2. checksums.txt：校验文件，二进制格式存储；它保存了 其余各类文件(primary.idx、count.txt等)的size大小以及size的哈希值，用于快速检验文件的完整性和正确性；
3. columns.txt：列信息文件，明文格式存储；保存列字段信息；
4. count.txt：计数文件，明文格式存储；记录当前分区数据的总条数；
5. primary.idx：一级索引文件，二进制格式存储；用于存放稀疏索引，一张MergeTree表只能只能声明一次一级索引(通过ORDER BY 或 PRIMARY KEY)。借助稀疏索引，在数据查询的时候能够排除主键 条件范围之外的数据文件，减少数据扫描范围，加快查询速度；
6. [Column].bin：数据文件，压缩格式存储，默认为LZ4压缩格式，用于存储某一列数据；由于MergeTree采用列式存储，所以每一列都拥有独立的.bin数据文件，并以列名命名；
7. [Column].mrk：列字段标记文件，二进制格式存储，保存了.bin文件中数据的偏移量；标记文件与稀疏索引对齐，又与.bin文件一一对应，每个字段都会拥有与其对应的.mrk标记文件，所以MergerTree通过标记文件建立了primary.idx稀疏索引与.bin熟据文件之间的映射关系；
-------
	稀疏索引(一级索引primary.idx) -> 标记文件(.mrk)[找到对应数据的偏移量] -> 数据文件(.bin)[通过偏移量读取数据]
-------
8. [Column].mrk2：如果使用了自适应大小的索引间隔，则标记文件会以.mrk2 命名，工作原理与.mrk文件相同；
9. partition.dat 与 minmax_[Column].idx：如果使用了分区键，例如PARTITION BY toYYYYMM(EventTime)，则会额外生成partition.dat和minmax索引文件，他们均是二进制格式存储；partition.dat用于保存当前分区的分区表达式最终生成的值；minmax索引文件用于标记当前分区的分区字段对应原始数据的最值；在这些分区索引的作用下，能够跳过不必要的数据分区目录，从而减少最终需要扫描的数据范围；
10. skp_idx_[Column].idx与skp_idx_[Column].mrk：如果声明了二级索引，则会生成相应的二级索引文件和标记文件，都使用二进制格式存储；二级索引在CK中又称为跳数索引，最终目标与一级索引相同，都是为了进一步减少所需扫描的数据范围，加速查询。

### 5.3 数据分区
- 在MergeTree中数据是以分区目录的形式组织的，数据分区(partition)与数据分片(shard)是不同的概念，数据分区是针对本地数据而言的，是对数据的一种纵向的切分;
- MT并不能依靠分区特性将一张表的数据分布存储到多个CK服务节点，而数据分片具有横向切分的能力；
#### 5.3.1 数据分区规则
| 类型     | 样例数据        | 分区表达式             | 分区ID                        |
|----------|-----------------|------------------------|-------------------------------|
| 无分区键 |                 | 无                     | all                           |
| 整型     | 18, 19, 20      | PARTITION BY Age       | 分区1:18  分区2:19   分区3:20 |
| 日期     | 2020-01-01      | PARTITION BY EventTime | 分区1: 20200101               |
| 其他     | ‘www.baidu.com’ | PARTITION BY Url       | 分区1:127129jkhas8239123n13   |
-----
> 如果分区键取值不是整型，也不是日期，例如String\Float等，则通过128位Hash算法取其Hash值作为分区ID。
> 多个ID之间是通过-连字符链接的。
-----
#### 5.3.2 分区目录的命名规则
PartitionID_MinBlockNum_MaxBlockNum_Level
- PartitionID：就是分区ID；
- MinBLockNum和MaxBlockNum：最小、大数据块编号；（BlockNum是一个表内自增的编号，从1开始累加）
- Level：合并层次，可以理解为某个分区被合并过的次数。
<br>
&nbsp; &nbsp; &nbsp; &nbsp; 20200101_1_1_0
----

#### 5.3.3 分区目录合并过程
- 分区目录是在数据写入过程中被创建的，伴随着新数据的写入(INSERT)，MergeTree会生成一批新的分区目录;
- 在之后的某个时刻(写入后10-15分钟，也可以手动执行optimize查询语句)，CK会通过后台任务将相同分区的多个目录合并成一个目录，目录中的索引和数据文件也会进行合并；
- 旧分区目录不会被立刻删除，而是在之后某个时刻由后台任务删除(默认8分钟)，但是旧分区目录已经不再是激活状态(active=0)，在数据查询时会自动被过滤掉。

新目录名称的合并方式遵循以下规则：
- MinBlockNum：取同一分区内所有目录中最小的MinBlockNum;
- MaxBlockNum：取同一分区内所有目录中最大的MaxBlcokNum;
- Level：取同一分区内最大Level值并加1.

### 5.4 一级索引
- MergeTree定义主键PRIMARY KEY之后，会依据index_granularity间隔(默认8192行)，为数据表生成一级索引并保存至primary.idx文件；
- 更为常见的简化形式是直接通过ORDER BY指代主键，此时PRIMARY KEY和ORDER BY定义相同，所以索引(primary.idx)和数据(.bin)会按照相同的规则排序。
#### 5.4.1 稀疏索引
- 一级索引primary.idx采用稀疏索引实现，稀疏索引是每一行索引标记对应一段数据；
- 稀疏索引占用空间小，所以primary.idx内的索引数据常驻内存，取用速度极快。
#### 5.4.2 索引粒度
- 索引粒度：index_granularity参数定义；
- MergeTree使用MarkRange表示一个具体的区间，通过start和end表示具体的范围。
- index_granularity不单只作用于一级索引(.idx)，同时也会影响数据标记(.mrk)和数据文件(.bin)。
- 因为仅有一级索引是完不成查询工作的，他需要借助数据标记才能定位数据，所以一级索引和数据标记的间隔粒度相同，彼此对齐；而数据文件也会按照index_granularity的间隔粒度进行数据压缩。
#### 5.4.3 索引数据生成规则
MergeTree对于稀疏索引数据的存储是很紧凑的，索引值前后相连，按照主键字段顺序紧密地排列在一起。  
#### 5.4.4 索引的查询过程
索引查询就是两个数值区间的交集判断：  
- 一个区间是由基于主键的查询条件转换而来的条件区间；
- 另一个区间是MarkRange的数值区间。
### 5.5 二级索引
- MergeTree同样也支持二级索引，又称为跳数索引，它是由数据的聚合信息构建而成, 它能够为非主键字段的查询发挥作用；
- 默认是关闭的，需要设置allow_experimental_data_skipping_indices(新版本中已经取消)才能使用；
- 跳数索引需要在CREATE语句中定义，它支持使用 元组和表达式的形式声明，完整语法如下：
	INDEX index_name expr TYPE index_type(...) GRANULARITY granularity
- 与一级索引一样，如果在建表语句中声明了跳数索引，则会额外生成相应的索引文件(skp_idx_[Column].idx)和标记文件(skp_idx_[Column].mrk);

#### 5.5.1 granularity和index_granularity的关系
不同跳数索引之间，除了它们自身独有的参数index_granularity之外，还有一个共同参数granularity:  
- index_granularity定义的是数据的粒度；
- granularity定义了聚合信息汇总的粒度；  
（换言之，granularity定义了一行跳数索引能够跳过多少个index_granularity区间的数据；GRANULARITY N 中的N就是设定二级索引对一级索引粒度的粒度。）

跳数索引数据的生成规则：  
- 首先，按照index_grannularity粒度间隔将数据划分成n段，总共有[0, n-1]个区间(n=total_rows/index_granularity, 向上取整)；
- 其次，根据跳数索引定义时声明的表达式，从0区间开始依次按照index_granularity粒度从数据中获取聚合信息，每次向前移动1步(n+1)，聚合信息逐步累加；
- 最后，当移动granularity次区间时，则汇总并生成一行跳数索引数据。<br>
![ck_granularity](./ck_granularity.jpg)

如上图，index_granularity=8192且granularity=3，则数据会按照index_granularity划分成n等份，  
MergeTree从第0段分区开始，以此获取聚合信息；当获取到第3个分区时(granularity=3)，则汇总并会生成第一行跳数索引数据。

#### 5.5.2 跳数索引类型
目前MergeTree共支持4种跳数索引：minmax, set, ngrambf_v1 和 tokenbf_v1。（一张表可以同时支持多个跳数索引）  

CREATE TABLE skip_test(  
&nbsp; &nbsp; &nbsp; &nbsp; ID string,  
&nbsp; &nbsp; &nbsp; &nbsp; Url String,  
&nbsp; &nbsp; &nbsp; &nbsp; Code String,   
&nbsp; &nbsp; &nbsp; &nbsp; EventTiem Date,  
&nbsp; &nbsp; &nbsp; &nbsp; INDEX a ID TYPE minmax GRANULARITY 5,  
&nbsp; &nbsp; &nbsp; &nbsp; INDEX b (length(ID) * 8) TYPE set(2) GRANULARITY 5,  
&nbsp; &nbsp; &nbsp; &nbsp; INDEX c (ID, Code) TYPE ngrambf_v1(3, 256, 2, 0) GRANULARITY 5,  
&nbsp; &nbsp; &nbsp; &nbsp; INDEX d ID TYPE tokenbf_v1(256, 2, 0) GRANULARITY 5  
)ENGINE=MergeTree()  
....  

### 5.6 数据存储
#### 5.6.1 各列独立存储
在MergeTree中，数据是按列存储的；每个列对应一个.bin文件，数据文件是以分区目录的形式被组织存放的，所以在.bin文件中只会保存当前分区片段内的这一部分数据。  
独立存储的优势：  
- 可以更好地进行压缩（相同类型数据放在一起）
- 能够最小化数据扫描范围
--------
MergeTree将数据写入.bin文件的方式：  
- 首先，数据经过压缩，默认LZ4压缩算法；
- 其次，数据会按照ORDER BY的声明排序；
- 最后，数据以压缩数据块的形式被组织并写入.bin文件中。

#### 5.6.2 压缩数据块
- 一个压缩数据块由头信息和压缩数据两部分构成。
- 头信息占9个字节，由1个UInt8(1字节)整型和2个UInt32(4字节)整型组成，分别代表：使用的压缩算法类型、压缩后数据大小 和 压缩前数据大小。<br>
![压缩数据块](./compressed_data.jpg)

(CK提供的clickhouse-compressor工具能够查询某个.bin文件中压缩数据的统计信息)  
- 每个压缩数据块的体积，按照其压缩之前的数据字节大小，都被严格控制在64KB-1MB之间，这个上下限分别由min_compress_size(默认65536)和max_compress_block_size(默认1048576)参数指定；
- 如果一个间隔(index_granularity)内数据大小 小于 64KB，则继续获取下一批次数据，直到累积到 size>=64KB，生成一个压缩数据块；如果一个间隔内数据 64KB<= size <= 1MB, 则直接被压缩成一个数据块；如果size>1MB, 则首先按照1MB截断并生成一个压缩数据块，剩余数据继续依照上述规则执行。

数据压缩目的：  
- 有效减小数据大小，降低存储空间并加速数据传输效率； 但数据压缩和解压动作会带来额外性能损耗，所以需要控制被压缩数据的大小，以求在性能损耗和压缩率之间寻求一种平衡；
- 在具体读取某列数据时(.bin文件)，首先加载压缩数据到内存，并解压；通过压缩数据块，可以在不读取整个.bin文件的情况下将读取粒度降低到压缩数据块级别，从而进一步缩小数据读取范围。

### 5.7 数据标记
数据标记为 一级索引(.idx)和数据文件(.bin)之间建立联系；  
数据标记记录了两个重要信息：  
- 记录一级索引的索引信息；
- 记录数据在数据块中的偏移量。

#### 5.7.1 数据标记生成规则
- 数据标记 和索引区间 是对齐的，均按照index-granularity的粒度间隔；所以只需要通过索引区间的下标就可以直接找到对应的数据标记编号；
- 数据标记文件也与数据文件一一对应，即每一列字段[Column].bin文件都有一个与之对应的[Column].mrk数据标记文件，用于记录数据在.bin文件中的起始偏移量。

一行*标记数据*使用一个元组表示一个片段的数据(默认8192行)，元组内包含两个整型数值，分别是：  
- 数据在.bin压缩文件中的 压缩数据块的起始偏移量；
- 数据在 解压数据块 中的起始偏移量。

*标记数据*与*一级索引*不同，它并不能常驻内存，而是使用LRU（最近最少使用）缓存策略加快其取用速度。  

#### 5.7.2 数据标记的工作方式
MergeTree引擎在查找数据时，整个过包括两个步骤：*读取压缩数据块*和*读取数据*;<br> 
![mrk](./mrk.jpg)

上图说明：  
前提：  
- JavaEnable字段数据类型为 UInt8,每行数值占用1字节，而数据表的index_granularity默认为8192；所以每一个索引片段的数据大小为8192B；
- 按照压缩数据块生成规则，如果单批数据小于64KB，则继续获取下一批数据，直至累积到size>=64KB,才会生成下一个压缩数据块；因此对于JavaEnable字段，8个索引片段对应的数据 会被压缩到一个压缩数据块中(1B * 8192=8192B, 64KB=65536B, 65536/8192=8);
- 也就是 每8行 标记数据 对应 一个 压缩数据块，所以这8行的 压缩文件的起始偏移量都是相同的，而它们在解压数据中的起始偏移量是按照8192B(每一个索引片段8192行数据)累加的。

MergeTree定位压缩数据块并读取数据：  
1. 读取压缩数据块：  
MergeTree查询某列数据时，不会加载整个.bin文件，而是根据需要只加载特定的压缩数据块：  
- 标记数据中 上下相邻的两个压缩文件中的起始偏移量构成了压缩数据块的偏移区间；  
	(例如:前8个索引片段 都会对应到.bin压缩数据文件中的[0, 12016]压缩数据块；这个区间是加上了前后压缩块的头信息的，头信息固定由9个字节组成，压缩后占8B。)  
- 压缩数据块被加载到内存后进行解压，之后进入具体数据的读取；  
2. 读取数据：  
MergeTree并不需要整段解压 压缩数据块，可以根据需要，以index_granularity的粒度加载特定的数据段：  
- 标记数据中 上下相邻的两个 解压数据的起始偏移量构成了 解压数据块的偏移区间；  
- 解压之后，依据解压数据块中的起始偏移量读取数据。  

### 5.8 对于分区、索引、标记和压缩数据的协同总结
#### 5.8.1 写入过程
1. 伴随着每一批数据的写入，都会生成新的分区目录；在后续某一时刻，相同分区的目录会合并；
2. 按照index_granularity索引粒度，会分别生成primary.idx一级索引(如果声明了二级索引，还会创建二级索引)、每一个列字段的.mrk数据标记和.bin压缩数据文件；

#### 5.8.2 查询过程
- 查询的本质可以看做一个不断缩小数据范围的过程；  
- 理想情况下MergeTree首先依次借助分区索引、一级索引和二级索引，将数据扫描范围缩至最小，然后借助标记数据，将需要解压和计算的数据范围缩至最小；
- 即使MergeTree不能预先减小数据范围，它会扫描所有的分区目录以及目录内索引短的最大区间，虽然不能减少数据范围，但是仍然能够借助数据标记，以多线程的形式同时读取多个压缩数据块，以提升性能。

## 6. MergeTree系列表引擎 
### 6.1 MergeTree
前面已经介绍MergeTree提供了数据分区、一级索引和二级索引等功能，在此进一步介绍MergeTree家族独有另外两项能力：数据TTL 与 存储策略。  

#### 6.1.1 数据TTL
- TTL Time to Live，数据的存活时间；MergeTree可以为某个字段或者整个表设置TTL，如果同时设置列级别和表级别TTL，则会以先到期的TTL为主;
- TTL需要依托某个DateTime或Date类型的字段，通过对这个字段;
- INTERVAL支持SECOND、MINUTE、HOUR、DAY、WEEK、MONTH、QUARTER 和 YEAR。

##### 6.1.1.1 列级别TTL

TTL到期之后，列值会被还原为对应数据类型的默认值。<br> 
CREATE TABLE ttl_table_v1( <br>
&nbsp; &nbsp; &nbsp; &nbsp; id String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime, <br>
&nbsp; &nbsp; &nbsp; &nbsp; code String TTL create_time + INTERVAL 10 SECOND, <br>
&nbsp; &nbsp; &nbsp; &nbsp; type UInt8 TTL create_time + INTERVAL 10 SECOND  <br>
)ENGINE=MergeTree()<br>
PARTITION BY toYYYYMM(create_time)  <br>
ORDER BY id  <br>
---------
- 修改列字段的TTL，或者添加列字段的TTL：  
- ALTER TABLE ttl_table_v1 MODIFY COLUMN code String TTL create_time + INTERVAL 1 DAY
- 目前CK没有提供取消列级别TTL的方法。

##### 6.1.1.2 表级别TTL
TTL到期之后，会将过期的数据行整行删除。<br> 
<br>
CREATE TABLE ttl_table_v2(<br>
&nbsp; &nbsp; &nbsp; &nbsp; id String, <br>
&nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime, <br>
&nbsp; &nbsp; &nbsp; &nbsp; code String TTL create_time + INTERVAL 1 MINUTE,<br>
&nbsp; &nbsp; &nbsp; &nbsp; type UInt8 <br>
)ENGINE=MergeTree()<br>
PARTITION BY toYYYYMM(create_time) <br>
ORDER BY create_time <br>
TTL create_time + INTERVAL 1 DAY <br> 

- 修改表的TTL：  
- ALERT TABLE ttl_table_v2 MODIFY TTL create_time + INTERVAL 3 DAY
- 表级别TTL目前也没有取消方法。

##### 6.1.1.3 TTL运行机理  
- 如果设置了TTL，在写入数据时，会以在分区目录下创建ttl.txt文件；ttl.txt文件通过JSON配置保存TTL的相关信息：columns用于保存列级别TTL信息，table由于保存表级别TTL信息。
- 只有在合并分区时，才会触发删除TTL过期数据的逻辑；
- 在选择删除分区时，会使用贪婪算法，算法规则是尽可能找到会最早过期的，同时年纪又是最老的分区（合并次数更多，MaxBlockNum更大的）；
- 一个分区中某列数据过期，合并之后的新分区目录中将不会包含这个字段的数据文件(.bin和.mrk)
- TTL默认的合并频率由MergeTree的merge_with_ttl_timeout参数控制，默认86400秒，即1天。它维护的是一个转悠的TTL任务队列，有别于MergeTree的常规合并任务，如果这个值被设置的过小，可能会带来性能的损耗；
- 可以使用optimize命令强制触发合并：  
	&nbsp; &nbsp; &nbsp; &nbsp; optimize TABLE table_name  //触发一个分区的合并<br> 
	&nbsp; &nbsp; &nbsp; &nbsp; optimize TABLE table_name FINAL   //触发所有分区合并<br>
- CK虽然没有提供删除TTL的方法，但是提供了控制全局TTL合并任务的启停方法：  
	SYSTEM STOP/START TTL MERGES      //还是不能做到按每张数据表启停  

#### 6.1.2 多路径存储策略
目前有三种存储策略：  
1. 默认策略：所有分区会自动保存到config.xml配置中的path指定路径下；
2. JBOD策略：Just a Bunch of Disks，是一种轮询策略；这种策略效果类似RAID 0，可以降低单块磁盘的负载，在一定条件下能够增加数据并行读写的性能；
3. HOT/COLD策略：适合服务器挂载了不同类型磁盘的场景；

### 6.2 ReplacingMergeTree
- MergeTree拥有主键，但是它的主键没有唯一键的约束，这就意味着即便多行数据的主键相同，他们还是能够被正常写入的；
- ReplacingMergeTree则能够在合并分区的时候删除重复数据，确实也在‘一定程度’上解决了重复数据的问题。（以分区为单位删除重复数据）;
- 创建：ENGINE=ReplacingMergeTree(ver)   //ver是选填，会指定一个UInt*、Date或者DateTime类型的字段作为版本号，这个参数决定了数据去重时所使用的算法。  
  
CREATE TABLE replace_table( <br>
&nbsp; &nbsp; &nbsp; &nbsp; id String, <br>
&nbsp; &nbsp; &nbsp; &nbsp; code String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime<br>
)ENGINE=ReplacingMergeTree(create_time)<br>
PARTITION BY toYYYYMM(create_time) <br>
ORDER BY (id, code) <br>
PRIMARY KEY id <br>

----
- ORDER BY 所声明的表达式是后续作为判断数据是否重复的依据；
- 只有在合并分区的时候才会触发删除重复数据的逻辑；
- 分区合并时，同一分区的重复数据才会被删除，不同分区的重复数据不会被删除；
- 分区内数据已经基于ORDER BY进行了排序，所以能够找到那些相邻的重复数据；
- 数据去重有两种策略：
 	1. 如果没有设置ver版本号，则保留同一分区中最后一行的重复数据；
	2. 如果设置了ver版本号，则保留同一分区中ver字段取值最大的的那一行。
-----

### 6.3 SummingMergeTree
- 场景：终端用户不关心明细数据，只查询数据汇总结果，并且汇总条件预先明确(GROUP BY条件明确，且不会随意改变)
- 直接查询存在问题：
	1. 额外存储开销：终端用户不关心明细数据，所以不应该一直保存所有明细数据；
	2. 额外查询开销：每次查询都进行实时聚合计算会有性能损耗。
- 解决：SummingMergeTree能够在每次合并分区时按照预先定义的条件进行数据的聚合计算，既减少了数据条数，又降低了后续查询开销：
-----
- MergeTree在每个数据分区中会按照ORDER BY表达式排序，主键索引也会按照PRIMARY KEY表达式取值并排序；而ORDER BY可以指代主键，所以一般情况下，只单独声明ORDER BY即可，此时ORDER BY和PRIMARY KEY定义相同，数据排序和主键索引相同。
- 如果同时定义了ORDER BY和PRIMARY KEY，那便是明确希望它俩不同；同时声明时，MergeTree会强制要求PRIMARY KEY字段必须是ORDER BY的前缀。
-----

SummingMergeTree使用:<br>
CREATE TABLE  summing_table(<br>
&nbsp; &nbsp; &nbsp; &nbsp; id String, <br>
&nbsp; &nbsp; &nbsp; &nbsp; city, String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; v1 UInt32, <br>
&nbsp; &nbsp; &nbsp; &nbsp; v2 Float64, <br>
&nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime<br>
)ENGINE=SummingMergeTree()<br>
PARTITION BY toYYYYMM(create_time)<br>
ORDER BY (id, city)<br>
PRIMARY KEY id<br>

//ENGINE=SummingMergeTree((col1, col2, ...)) 	参数是选填的，用于设置除主键之外的其他**数值类型字段**，以指定被SUM汇总的列字段;<br> 
//如果不填参数，则会将所有 非主键**数值类型字段**进行SUM汇总.<br>

SummingMergeTree处理逻辑：<br>
1. 用ORDER BY排序键作为聚合数据的条件KEY；
2. 只有在合并分区时才会触发汇总的逻辑；
3. 以分区为单位进行汇总；
4. 在进行数据汇总时，因为分区内数据已经是基于ORDER BY排序，所以能够找到相邻且拥有相同聚合KEY的数据；
5. 对于汇总字段会进行SUM计算，对于非汇总字段则会取第一行数据；
6. **支持嵌套结构，但列字段名称必须以Map作为后缀；嵌套类型中默认以第一个字段作为聚合Key，除第一个字段外，任何名称以Key、Id或Type为后缀的字段都将和第一个字段一起组成复合Key。**

### 6.4 AggregatingMergeTree
- 它是SummingMergeTree的升级版，但是可以自定义聚合函数；
- ENGINE=AggregatingMergeTree()   *//没有任何参数；在分区合并时会按照ORDER BY聚合；可以通过AggregateFunction来定义聚合函数；*

-----
CREATE TABLE agg_table(<br>
&nbsp; &nbsp; &nbsp; &nbsp; id String, <br>
&nbsp; &nbsp; &nbsp; &nbsp; city String, <br>
&nbsp; &nbsp; &nbsp; &nbsp; code AggregateFunction(uniq, String),<br>
&nbsp; &nbsp; &nbsp; &nbsp; value AggregateFunction(sum, UInt32),<br>
&nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime <br>
)ENGINE=AggregatingMergeTree()<br>
PARTITION BY toYYYYMM(create_time)<br>
ORDER BY (id, city) <br>
PRIMARY KEY id <br>
------

- AggregateFunction 是CK提供的一种特殊的数据类型，它能够以二进制形式存储中间状态结果；写入数据时需要调用 *State函数，读取数据时需要调用相应的 *Merge函数：
- **写入数据** <br>
INSERT INTO TABLE agg_table<br>
SELECT 'A00', 'wuhan', <br>
uniqState('code1'), <br>
sumState(toUInt32(100)),<br>
'`2020-01-01 00:00:01`' <br>

- **查询数据** <br>
SELECT id, city, uniqMerge(code), sumMerge(value)FROM agg_table<br> 
GROUP BY id, city <br>

- 上述正常情况下过于复杂，AggregatingMergeTree更为常见的应用方式是结合 *物化视图* 使用，即将它作为物化视图的表引擎：
- **底表** <br>
//用于存储全量的明细数据，并以此对外提供实时查询。<br>
CREATE TABLE agg_table_basic(<br>
&nbsp; &nbsp; &nbsp; &nbsp; id String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; city String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; code String,<br>
&nbsp; &nbsp; &nbsp; &nbsp; value UInt32 <br>
)ENGINE=MergeTree() <br>
PARTITION BY city <br>
ORDER BY (id, city) <br>
 <br>
- **物化视图**<br>
// 用于特定场景的数据查询，相比MergeTree他拥有更高的性能。<br>
CREATE MATERIALIZED VIEW agg_view<br>
ENGINE=AggregatingMergeTree() <br>
PARTITION BY city <br>
ORDER BY (id, city) <br>
AS SELECT <br>
&nbsp; &nbsp; &nbsp; &nbsp; id, <br>
&nbsp; &nbsp; &nbsp; &nbsp; city,<br> 
&nbsp; &nbsp; &nbsp; &nbsp; uniqState(code) AS code,<br>
&nbsp; &nbsp; &nbsp; &nbsp; sumState(value) AS value<br>
FROM agg_table_basic <br>
GROUP BY id, city <br>

- 新增数据时，面向的对象是底表MergeTree：<br>
INSERT INTO TABLE agg_table_basic<br>
VALUES('A001', 'wuhan', 'code1', 100), ('A002', 'bejing', 'code2', 200)<br>
//数据会自动同步到 *物化视图*, 并按照定义的规则处理数据:<br>
- 查询：<br>
	> SELECT id, *sumMerge*(value), *uniqMerge*(code), FROM agg_view GROUP BY id, city <br>
-----
AggregatingMergeTree的处理逻辑：<br>
1. 用ORDER BY 排序键作为聚合数据的条件KEY；
2. 使用AggregateFunction字段类型定义聚合函数的类型以及聚合的字段；
3. 只有在合并分区是才会触发聚合计算的逻辑；
4. 以数据分区作为聚合单位；
5. 进行数据计算时，分区内数据已经是基于ORDER BY排序，所以能够找到相邻且拥有相同聚合KEY的数据；
6. 聚合数据时，同一分区拥有相同分区Key的数据会被合并成一行，对于非主键、非AggregateFuntion字段，则会取用第一行数据；
7. AggregateFuntion字段采用二进制存储，在写入数据时需先调用*State函数，读取数据时需调用*Merge函数，*表示定义时使用的聚合函数；
8. AggregatingMergeTree通常作为物化视图的表引擎，与普通MergeTree配合使用。

### 6.5 CollapsingMergeTree
- 对于CK的数据源文件修改是代价很高的，相较于直接修改源文件，它会将修改和删除操作转化为 *新增操作*，即以增代删;<br>
- CollapsingMergeTree(折叠合并树)就是一种以增代删的思路，它支持行级的数据修改和删除的表引擎；<br>
- 它通过定义一个sign标记位字段(Int8)来记录数据行的状态，sign=1表示有效，sign=-1表示被删除；<br>
- 当CollapsingMergeTree合并分区时，同一分区内，sign标记为1和-1 的一组数据会被抵消删除，犹如一张瓦楞纸折叠一般；<br>
----
实例：<br>
CREATE TABLE collapse_table(<br>
&nbsp; &nbsp; &nbsp; &nbsp;id String, <br>
&nbsp; &nbsp; &nbsp; &nbsp;code Int32, <br>
&nbsp; &nbsp; &nbsp; &nbsp;create_time DateTime, <br>
&nbsp; &nbsp; &nbsp; &nbsp;sign Int8 <br>
)ENGINE=CollapsingMergeTree(sign)<br>
PARTITION BY toYYYYMM(create_time)<br>
ORDER BY id<br>

- 与其他MergeTree变种一样，CollapsingMergTree同样是以ORDER BY排序键作为后续判断数据唯一性的依据;<br>
- 修改操作：( **ORDER BY字段与原始数据相同(其他字段可以不同)，sign取反-1**)
	- 修改前原始数据插入：INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *1*)
	- 修改操作：
		- 先标记原始数据失效：INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *-1*)
		- 再添加修改数据： &nbsp; &nbsp; INSERT INTO TABLE collapse_table VALUES('A001', *920*, '`2020-01-02 00:00:00`', *1*)
- 删除操作：( *ORDER BY字段与原始数据相同，sign取反-1*)
	- 删除前原始数据插入：INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *1*)
	- 删除操作：&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *-1*)
<br>
- 注意事项：
	1. 折叠数据不是实时触发的，也是在合并分区时才会触发( **所以在合并之前用户还是会看到旧数据**)；
		- 要么查询前，手动执行optimize TABLE table_name FINAL 触发分区合并；
		- 要么修改SQL语句：
			- 原始SQL：SELECT id, SUM(code), COUNT(code), AVG(code), UNIQ(code) FROM collapse_table GROUP BY id
			- 现在SQL：SELECT id, SUM(code * sign), COUNT(code * sign), AVG(code * sign), uniq(code * sign) FROM collapse_table GROUP BY id HAVING SUM(sign)>0
	2. 只有相同分区内数据才会被折叠；
	3. 它是基于ORDER BY排序的，对于写入数据的顺序有严格要求：
		- 正常先写sign=1，再写sign=-1；如果先写sign=-1，再写sign=1，则不能够折叠。
		- 如果写入程序是多线程，就不能保证数据的写入顺序，CollapsingMergeTree的工作机制就会出问题。

### 6.6 VersionedCollapsingMergeTree
- 它的作用与CollapsingMergeTree完全相同，不同之处是它对数据写入顺序没有要求；
- 在同一分区内，任意顺序的数据都能够完成折叠操作，依据是版本号。

-----
- 实例：
CREATE TABLE ver_collapse_table(<br>
 &nbsp; &nbsp; &nbsp; &nbsp; id String, <br> 
 &nbsp; &nbsp; &nbsp; &nbsp; code Int32, <br>
 &nbsp; &nbsp; &nbsp; &nbsp; create_time DateTime, <br>
 &nbsp; &nbsp; &nbsp; &nbsp; sign Int8, <br>
 &nbsp; &nbsp; &nbsp; &nbsp; ver UInt8<br>
)ENGINE=VersionCollapsingMergeTree(sign, ver)<br>
PARTITION BY toYYYYMM(create_time)<br>
ORDER BY id<br>
-----
- VersionCollapsingMergeTre是如何利用版本号字段的呢？
	- 定义ver字段后，它会自动将ver字段作为排序条件，上述实例排序是：ORDER BY id, ver DESC 
	- 所以无论写入顺序如何，折叠时都能够回到正确顺序。
----
- 修改前原始数据插入：INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *1, 1*)
- 修改操作：
	- 先标记原始数据失效：INSERT INTO TABLE collapse_table VALUES('A001', 100, '`2020-01-02 00:00:00`', *-1, 1*)
	- 再添加修改数据： &nbsp; &nbsp; INSERT INTO TABLE collapse_table VALUES('A001', 920, '`2020-01-02 00:00:00`', *1, 2*)

### 6.7 各种MergeTree之间的关系总结
![all_mergetree](./all_mergetree.jpg) <br>

- 除了MergeTree之外的其他6个变种表引擎的Merge合并逻辑，全都是建立在MergeTree基础之上的，且均继承于MergeTree的MergingSortedBlockInputStream;
- MergingSortedBlockInputStream的主要作用是按照ORDER BY的规则保持新分区数据的有序性。

## 7. 副本与分片
- 副本主要是防止数据丢失，增加数据存储的冗余；
- 分片主要是实现数据的水平切分。

### 7.1 ReplicatedMergeTree
- 此前介绍了7中MergeTree，此处介绍ReplicatedMergeTree系列。ReplicatedMergeTree于普通MergeTree有什么区别？
 ![replicated](./replicated.jpg)

 ![replicated_merge](./replicated_merge.jpg)

- ReplicatedMergeTree在MergeTree能力的基础上增加了分布式协同的能力，其借助ZooKeeper的消息日志广播功能，实现了副本实例之间的数据同步功能；
- 只有使用了ReplicatedMergeTree复制表系列引擎，才能应用副本的能力（即使用ReplicatedMergeTree的数据表就是副本）；
---------
在MergeTree中，一个数据分区由开始创建到完成写入，会经历两类存储区域：<br>
- 内存：数据会被先写入内存缓冲区；
- 本地磁盘：接着数据会被写入tmp临时分区目录，等待完成写入后再将临时目录重命名为正式分区目录。

ReplicatedMergeTree在上述基础上增加了ZooKeeper部分，它会进一步在ZooKeeper中创建一系列监听节点，并以此实现多个节点之间的通信；<br>
**但是，在整个通信过程中，ZooKeeper并不会涉及表数据的传输。**<br>
 
### 7.2 ReplicatedMergeTree特点
- 依赖ZooKeeper：只有在执行INSERT和ALTER操作的时候，才需要借助ZooKeeper的分布式协同能力，实现多个副本之间的同步；在查询副本的时候不需要ZooKeeper；
- 表级别的副本：可以自定义副本数量，以及副本 在集群内的分布位置；
- 多主架构(Multi Master)：可以在任意副本上执行INSERT和ALERT操作，这些操作会借助ZooKeeper的协同能力被分发到每个副本上以本地形式执行；
- Block数据块：在执行INSERT写入数据时，会依据max_insert_block_size(默认1048576行)将数据切分成若干个Block数据块，所以数据块是数据写入的基本单元，并且具有写入的原子性和唯一性；
- 原子性：写入数据时，以Block为操作单元；
- 唯一性：写入数据块时，它会按照Block的数据顺序、数据行和数据大小等指标，计算Hash信息摘要并记录在案；如果后续有相同Hash摘要信息的Block写入，则会被忽略，可以预防Block被重复写入。

### 7.3 副本定义
- 定义方式：ENGINE=ReplicatedMergeTree('zk_path', 'replica_name')
	- zk_path：用于指定在ZooKeeper中创建的数据表路径，可以参考：/clickhouse/tables/{shard}/table_name //(shard表示分片的编号01、02)；
	- replica_name：用于定义在ZooKeeper中创建的副本名称，该名称是区别副本实例的唯一标识；通常命名方式是使用所在服务器的域名命名；
----
- 多分片、一个副本的情形：
	//分片1<br>
	ReplicatedMergeTree('/clickhouse/tables/01/test_1', 'ch5.nauu.com')<br>
	ReplicatedMergeTree('/clickhouse/tables/01/test_1', 'ch6.nauu.com')<br>
	
	//分片2<br>
	ReplicatedMergeTree('/clickhouse/tables/02/test_1', 'ch7.nauu.com')<br>
	ReplicatedMergeTree('/clickhouse/tables/02/test_1', 'ch8.nauu.com')<br>
----
### 7.4 ReplicatedMergeTree原理解析
#### 7.4.1 数据结构
- ReplicatedMergeTree核心逻辑大量运用了ZooKeeper的能力，以实现多ReplicatedMergeTree副本实例之间的协同，包括主副本选举、副本状态感知、操作日志分发、任务队列和BlockID去重判断等；
- 在执行 INSERT数据写入、MERGE分区合并 和 MUTATION修改操作的时候，都会涉及与ZooKeeper的通信，但通信过程不涉及表数据的传输；
- 在执行查询操作时不会访问ZooKeeper，所以不必过于担心ZooKeeper的承载能力。
------------
##### 7.4.1.1 ZooKeeper内的节点结构
- ReplicatedMergeTree是依靠ZooKeeper的 *事件监听机制* 来实现各个副本之间的协同，所以，在表创建过程中它会以zk_path为根路径，创建这张表的一组监听节点：
	1. 元数据
		- /metadata ：保存元数据信息，包括主键、分区键、采样表达式等；
		- /columns ：保存字段信息，包括列名称、数据类型；
		- /replicas ：保存副本名称，对应设置参数replica_name;
	2. 判断标识
		- /leader_election：用于主副本选举；
		- /blocks：记录Block的Hash信息摘要以及partition_id；（ *通过Hash摘要信息判断Block数据块是否重复；通过partition_id找到需要同步的数据分区;* ）
		- /block_numbers：按照分区的写入顺序，以相同的顺序记录partition_id；各个副本在其本地MERGE时，都会依据相同的block_numbers进行；
		- /quorum：（下限）当至少有quorum数量的副本写入成功后，整个写入过程才算成功；( *quorum由insert_quorum参数控制，默认为0；* )
	3. 操作日志
		- /log：常规操作日志节点(INSERT、MERGE、DROP PARTITION)，它是整个工作过程中最重要一环，保存了副本需要执行的指令；
			- log使用了ZK的持久顺序型节点；每一个副本实例都会监听/log节点，当有新指令加入时，它们会把指令加入到副本各自的任务队列，并执行任务；
		- /mutations：MUTATION操作日志节点，当执行ALERT、DELETE和ALERT UPDATE操作时，操作指令会被添加到这个节点；（也使用了ZK的持续顺序型节点）
		- /replicas/{replica_name}/*：每个副本各自节点下的一组监听节点，用于指导副本在本地执行具体的任务指令，较为重点的节点如下：
			- /queue：任务队列节点，用于执行具体的操作任务；当副本从/log或/mutations节点监听到操作指令时，会将执行任务添加到该节点下，并基于队列执行；
			- /log_pointer：log日志指针节点，记录了最后一次执行的log日志下标信息；
			- /mutation_pointer：mutations日志指针节点，记录了最后一次执行的mutations日志名称。
##### 7.4.1.2 Entry日志对象的数据结构
- 上述ReplicatedMergeTree在ZK中有两组非常重要的父节点：/log和/mutations；
- 它们作用犹如一座通信塔，是分发操作指令的信息通道，而发送指令的方式，则是为这些父节点添加子节点，各个副本实例都能实时感知；
--------
- 上述子节点在CK中被统一抽象为Entry对象，而具体实现则由LogEntry和MutationEntry对象承载，分别对应/log和/mutations节点：
	1. LogEntry：用于封装/log的子节点信息，核心属性：
		- source replica：发送这条Log指令的副本来源，对应replica_name;
		- type：操作指令类型，只要有get、merge和mutate三种，分别对应从远程副本下载分区、合并分区和MUTATION操作；
		- block_id：当前分区的BlockID，对应/blocks路径下子节点的名称；
		- partition_name：当前分区目录的名称；
	2. MutationEntry：由于封装/mutations的子节点信息，核心属性：
		- source replica：发送这条MUTATION指令的副本来源，对应replica_name;
		- commands：操作指令，主要有ALTER DELETE和ALTER UPDATE；
		- mutation_id：MUTATION操作的版本号；
		- partition_id：当前分区的ID；
------
#### 7.4.2 副本协同的核心流程
- 副本协同的核心流程主要有：INSERT(数据写入)、MERGE(分区合并)、MUTATION(数据修改)和ALTER(元数据修改)；
- INSERT和ALTER操作是分布式执行的，借助ZK的时间通知机制，多副本之间会自动进行协同，但是它们不会使用ZK存储任何分区数据；
- 其他操作并不支持分布式执行，包括SELECT、CREATE、DROP、RENAME和ATTACH；（例如为了创建多个副本，需要分别登录每个CK节点，并在它们本地各自执行CREATE操作）
1. INSERT的核心执行流程<br>
 ![insert](./insert.jpg)

按照示意图，大致可分为8个步骤：<br>
	1. 创建第一个副本实例
		首先在CH5节点上创建第一个副本实例：<br>
		CREATE TABLE replicated_sales_1(<br>
		&nbsp; &nbsp; id String,<br>
		&nbsp; &nbsp; price Float64,<br>
		&nbsp; &nbsp; create_time DateTime<br>
		) ENGINE=ReplicatedMergeTree('/clickhouse/tables/01/replicated_sales_1', 'ch5.nauu.com')<br>
		PRTITION BY toYYYYMM(create_time)<br>
		ORDER BY id<br>
		- 创建过程中，ReplicatedMergeTree会进行一些初始化操作：
			- 根据zk_path初始化所有的ZK节点；
			- 在/replicas/节点下注册自己的副本实例ch5.nauu.com;
			- 启动监听任务，监听/log日志节点；
			- 参与副本选举，选出主副本，选举方式是向/leader_election/插入子节点，第一个插入成功的副本就是主副本。
	2. 创建第二个副本实例
		在CH6节点下创建第二个副本实例：<br>
		CREATE TABLE replicated_sales_1(<br>
		ReplicatedMergeTree&nbsp; &nbsp; id String,<br>
		&nbsp; &nbsp; price Float64,<br>
		&nbsp; &nbsp; create_time DateTime<br>
		) ENGINE=ReplicatedMergeTree('/clickhouse/tables/01/replicated_sales_1', 'ch6.nauu.com')<br>
		PRTITION BY toYYYYMM(create_time)<br>
		ORDER BY id<br>
		- 创建过程中，第二个ReplicatedMergeTree会进行一些初始化操作：
			- 在/replicas/节点下注册自己的副本实例ch6.nauu.com;
			- 启动监听任务，监听/log日志节点；
			- 参与副本选举，选出主副本。
	3. 向第一个副本实例写入数据
		执行命令：INSERT INTO TABLE replicated_sales_1 VALUES('A001', 100, `'2020-01-01 00:00:00'`)<br>
		- 上述命令执行之后：
			- 首先，会在本地完成分区目录的写入：Renaming temporary part tmp_insert_202001_1_1_0 to 202001_0_0_0
			- 接着向/blocks节点写入该数据分区的block_id：Wrote block with ID '202001_295581757877228717_1212312312312432'
				- 该block_id将作为后续重操作的判断依据，即副本会自动忽略该block_id的重复写入；
			- 如果设置了insert_quorum参数(默认为0)，并且insert_quorum》=2，则CH5会进一步监控已完成写入操作的副本个数，只有当写入副本个数大于等于insert_quorum时，整个写入操作才算成功。
	4. 由第一个副本实例推送Log日志
		- 上述步骤完成后，会继续执行了INSERT的副本向/log节点推送操作日志：此处操作type会是get；
		- 需要下载分区202001_0_0_0，其余所有副本都会基于Log日志以相同顺序执行命令;
	5. 第二个副本实例拉取Log日志
		- CH6副本会一直监听/log节点变化，CH6会触发日志拉取任务并更新log_pointer;
		- CH6拉取到LogEntry之后，并不会直接执行，而是将其转为任务对象并放置任务队列；（ *此处拉取的是一个LogEntry区间，因为可能会连续收到多个LogEntry*）
	6. 第二个副本实例向其他副本发起下载请求
		- CH6是基于/queue队列开始执行任务的，当看到get类型的任务时，ReplicatedMergeTree会明白此时远端的其他副本中已经有成功写入的数据分区，而自己需要去同步这份数据；
		- CH6会选择一个远端的副本作为数据下载来源，选择算法大致为：
			- 从/replicas节点拿到所有的副本节点；
			- 遍历这些副本，选取log_pointer下标最大、/queue子节点数最少的副本；（log_pointer 下标最大意味着该副本执行的日志最多；/queue子节点数最少意味着该副本目前任务执行负担最小。）
			- 如果第一次请求失败，默认总请求次数为5(由max_fetch_partition_retries_count参数控制)。
	7. 第一个副本实例响应数据下载请求
		CH5的DataPartsExchange端口收到调用请求，根据参数做出响应。<br>
	8. 第二副本实例下载数据并完成本地写入
		- 收到CH5数据后，先写到临时目录，完成后再重命名该目录202001_0_0_0；
		- ZK不负责表数据的传输，而是副本实例之间点对点地下载分区数据；
2. Merge的核心执行流程
<br>
![merge](./merge.jpg)

Merge过程大致分为5个步骤：<br>
1. 创建远程连接，尝试与主副本通信
	- 首先在CH6执行OPTIMIZE命令强制触发MERGE合并；
	- CH6通过/replicas找到CH5，并尝试建立远程连接。
2. 主副本接受连接请求
	- 主副本CH5收到CH6的连接请求；
3. 由主副本指定MERGE计划并推送Log日志
	- CH5的合并计划会转换为Log日志对象并推送Log日志，以通知所有副本开始合并；
	- 同时，主副本还会锁住执行线程，对日志的接受情况进行监听；
		- 其监听行为由replication_alter_partitions_sync参数控制，默认为1；参数为0时，不做任何等待；参数为1时，只等待主副本自身完成；参数为2时，会等待所有副本拉取完成。
4. 各个副本分别拉取Log日志
	- 各个副本监听并拉取Log日志，然后将日志对象转为任务对象并放置任务队列；
5. 各个副本分别在本地执行MERGE
	- 各个副本实例基于任务队列开始执行任务；
----
- ZK不会进行实质性的数据传输；
- 无论MERGE操作在哪个副本被触发，都会先被转交至主副本，再由主副本完成 合并计划制定、消息日志的推送、日志接受情况的监控。
----

3. MUTATION的核心执行流程
- 当ReplicatedMergeTree执行ALTER DELETE 或者 ALTER UPDATE 操作时，才会进入MOTATION部分的逻辑；
- 5个步骤的流程如图：<br>
	![mutation](./mutation.jpg)
	
	1. 推送MUTATION日志
		ALTER TABLE replicated_sales_1 DELETE WHERE id='1'<br>
		- 创建MUTATION ID：create mutation with ID 0000000000
		- 将MUTATION操作转换为MutationEntry日志，并推送到/mutations/0000000000；（由此可知MUTATION日志是由/mutations节点分发给各个副本）
	2. 所有副本实例各自监听MUTATION日志
		- CH5、CH6都会实时监控/mutations节点；当监听到有新MUTATION日志加入时，它们首先会判断自己是否是主副本。
	3. 由主副本实例响应MUTATION日志并推送Log日志
		- 只有主副本才会响应MUTATION日志，CH5是主副本，它会将MUTATION日志转换为LogEntry日志并推送至/log节点，以通知各个副本执行具体的操作；
	4. 各个副本实例分别拉取Log日志
		- CH5、CH6分别监听Log日志推送，它们会分别拉取日志到本地，生成任务推送至/queue任务队列；
	5. 各个副本实例分别在本地执行MUTATION
------
- MUTATION执行过程中，ZK同样不会进行任何实质性的数据传输；
- MUTATION操作是经过/mutations节点实现分发的；
- 无论MUTATION操作从哪个副本被触发，之后都会被转交至主副本，再由主副本负责推送Log日志，以通知各个副本执行最终的MUTATION逻辑；
- 同时，也由主副本对日志接受情况实行监控。
------

4. ALTER的核心执行流程<br>
![alter](./alter.jpg)

ALERT操作是进行元数据修改，核心流程如下：<br>
	1. 修改共享元数据
		- CH6执行增加字段：ALTER TABLE replicated_sales_1 ADD COLUMN id2 String
		- 执行之后，CH6会修改ZK内的共享元数据节点；
		- 数据修改之后，节点版本号也会同时提升；同时CH6还会负责监听所有副本的修改完成情况。
	2. 监听共享元数据变更并各自执行本地修改
		- CH5、CH6分别监听共享元数据的变更，之后会分别与本地元数据版本号做比较；
		- 如果本地版本号低于共享版本号，则会在各自本地执行更新操作。
	3. 确认所有副本完成修改
		- 由CH6确认所有副本完成修改；
------
- ALTER操作过程中，CK也不会进行任何实质性的数据传输；
- 本着谁执行谁负责的原则，在此CH6负责对共享元数据的修改以及对各个副本修改进度的监控。
------

### 7.5 数据分片
- 引入数据副本，能够降低数据丢失风险，提升查询性能(分摊查询、读写分离)；要解决业务量庞大的情况，则需要引入分片(shard);
- CK的数据分片需要结合Distributed表引擎一同使用，Distributed表引擎自身不存储任何数据，它能够作为分布式表的一层透明代理；
#### 7.5.1 集群的配置方式
- 1分片、1副本的配置：
	```
	<shard> <!-- 分片 -->
		<replica> <!-- 副本 -->
		</replica>
		
		<replica>
		</replica>
	</shard>
	```
- shard 更像是逻辑层面的分组，而无论是副本还是分片，它们的载体都是replica，所以从某种角度讲，副本也是分片；

------
集群有两种配置形式：<br>
1. 不包含副本的分片
```
<yandex>
	<clickhouse_remote_servers>
		<shard_2> <!-- 自定义集群名称 -->
			<node>  <!-- 定义CK节点 -->
				<host>ch5.nauu.com</host>
				<port>9000</port>
			</node>
			<node>
				<host>ch6.nauu.com</host>
				<port>9000</port>
			</node>
		</shard_2>
	</clickhouse_remote_servers>
```
- shard_2表示自定义集群名称，全剧唯一；

2. 自定义分片和副本
- 集群sharding_ha 拥有2个分片，每个分片有1个副本：
```
<sharding_ha>
	<shard>
		<replica>
			<host>ch5.nauu.com</host>
			<port>9000</port>
		</replica>
		<replica>
			<host>ch6.nauu.com</host>
			<port>9000</port>
		</replica>
	</shard>
	<shard>
		<replica>
			<host>ch7.nauu.com</host>
			<port>9000</port>
		</replica>
		<replica>
			<host>ch8.nauu.com</host>
			<port>9000</port>
		</replica>
	</shard>
</sharding_ha>
```
- 集群中replica 数量的上限是由CK节点的数量决定的。

#### 7.5.2 基于集群实现分布式DDL
- 前面只有数据副本时，为了创建多张副本表，需要分别登录到每个CK节点，然后在本地各自执行CREATE语句；这是因为默认情况下，CREATE、DROP、RENAME和ALTER等DDL语句并不支持分布式执行；
- 加入集群配置后，就可以使用新的语法实现分布式DDL执行了，语法形式如下：
	- CREATE/DROP/RENAME/ALTER TABLE **ON CLUSTER cluster_name**
	- *//cluster_name对应了配置文件中的集群名称，CK会根据配置信息分别去各个节点执行DDL语句*
------
- 分布式DDL语句：
- 创建表：
```
	CREATE TABLE test_1_local ON CLUSTER shard_2(
		id UInt64
	)ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/test_1', '{replica}')
	ORDER BY id
	--这里可以使用其他表引擎
```
- **两个动态宏变量{shard}、{replica}是定义在各个节点配置文件中的。**

- 删除表：
```
	DROP TABLE test_1_local ON CLUSTER shard_2
```

