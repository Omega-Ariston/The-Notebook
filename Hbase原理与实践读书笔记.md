# HBase原理与实践
## 第1章 HBase概述
### 1.2 HBase数据模型
#### 1.2.1 逻辑视图
- HBase中的基本概念：
    - table：表，一个表包含多行数据
    - row：行，一行数据包含一个唯一标识rowkey、多个column以及对应的值。表中所有row都按照rowkey的字典序从小到大排序
    - column：列，与关系型数据库中的列不同，column由column family（列簇）以及qualifier（列名）两部分组成，中间使用“：”相连。column family在表他去的时候需要指定，用户不能随意增减。而一个column family下的qualifier可以设置任意多个，因此列可以动态增加，理论上可以扩展到上百万列
    - timestamp：时间戳，每个cell在写入HBase的时候都会默认分配一个时间戳作为该cell的版本，同一个rowkey、column下可以有多个value，版本号越大表示数据越新
    - cell：单元格，由五元组（row，column，timestamp，type，value）组成的结构，type表示put/delete这样的操作类型，鼻梁骨stamp表示版本。这个结构是以KV结构存储的，前四个元素组成的元组为key，value为value
- 总体而言，HBase引入了列簇的概念，列簇下的列可以动态扩展，时间戳则用以实现了数据的多版本支持

#### 1.2.2 多维稀疏排序Map
- 多维：HBase中的Map的key是一个复合数据结构，由多维元素构成（四元组）
- 稀疏：对于HBase，空值无需填充null，这是列可以无限扩展的一个重要条件
- 排序：构成HBase的KV在同一个文件中都是有序的，排序会分别依照rowkey、column family：qualifier、timestamp进行，这对提升HBase的读取性能至关重要
- 分布式：构成HBase的所有Map并不集中在某台机器上

#### 1.2.3 物理视图
- 与大多数数据库系统不同，HBase中的数据是按照列簇存储的，即将数据按照列簇分别存储在不同的目录中

#### 1.2.4 行式存储、列式存储、列簇式存储
- 行式存储：将一行数据存储在一起，写完一行再写下一行，如传统关系型数据库。但在查找列数据的过程中会引入无用列信息，从而导致多余内存占用，因此仅适合用于处理OLTP类型负载，对于OLAP类分析型负载并不擅长
- 列式存储：理论上会将一列数据存储在一起，不同列的数据分别集中存储，如Kudu、Parquet。同一列的数据通常具有相同数据类型，因此列式存储具有天然的高压缩特性
- 列簇式存储：介于前两者之间，可以通过不同的设计思路在两者间进行转换。比如一张表只设置一个列簇包含所有列，HBase中一个列簇的数据存在一起，此时就相当于行式存储。再比如一张表设置大量列簇，每个列簇下仅有一列，就相当于列式存储

### 1.3 HBase体系结构
#### 1. HBase客户端
- HBase客户端提供了Shell命令行接口、原生Java API编程接口、Thrift/REST API编程接口以及MapReduce编程接口。其中Thrift/REST API主要用于支持非Java的上层业务需求，MapReduce接口主要用于批量数据导入以及批量数据读取
- HBase客户端访问数据行之前会先通过元数据表定位目标数据所在RegionServer，之后都会发送请求到该RegionServer。同时元数据会被缓存在客户端本地。如果集群RegionServer发生了宕机或执行了负载均衡等，导致数据分片发生迁移，客户端需要重新请求最新的元数据并缓存在本地
#### 2. Zookeeper
- Zookeeper在HBase系统中扮演着非常重要的角色：
    - 实现Master高可用：Zookeeper会检测到Active Master的宕机事件，并通过一定机制选举出新的Master，保证系统正常运转
    - 管理系统核心元数据：比如，管理当前系统中正常工作的RegionServer集合，保存系统元数据表hbase:meta所在的RegionServer地址等
    - 参与RegionServer宕机恢复：ZooKeeper通过心跳可以感知到RegionServer是否宕机，并在宕机后通知Master进行宕机处理
    - 实现分布式表锁：HBase中对一张表进行各种管理操作，如alter时需要先加表锁，防止其它用户对同一张表进行管理操作造成表状态不一致
#### 3. Master
- 主要负责HBase系统的各种管理工作：
    - 处理用户的各种管理请求，比如建表、修改表、权限操作、切分表、合并数据分片以及Compaction等
    - 管理集群中所有RegionServer，包括RegionServer中Region的负载均衡、RegionServer的宕机恢复以及Region的迁移等
    - 清理过期日志以及文件，Master隔一段时间会检查HDFS中HLog是否过期、是否已经被删除，并在过期后将其删除
#### 4. RegionServer
- 主要用来响应用户的IO请求，是HBase中最核心的模块，由WAL（HLog）、BlockCache以及多个Region构成
    - WAL：有两个核心作用，其一是用于实现数据的高可靠性，HBase数据随机写入时先写入缓存再异步刷新落盘，而写入缓存之前需要先顺序写入HLog，这样即使缓存数据丢失也可以通过HLog日志恢复；其二是用于实现HBase集群间主从复制（通过回放HLog日志）
    - BlockCache：HBase系统中的读缓存。客户端从磁盘读取数据后通常会将数据缓存到系统内存中，后续访问同一行数据可以直接从内存中获取而不需要访问磁盘。
        - BlockCache缓存是一系列Block块，每个默认为64K，由物理上相邻的多个KV数据组成。BlockCache同时利用了**时间局部性**（访问的数据近期可能再次被访问）和**空间局部性**（缓存单位是block，而不是单个KV）原理。
        - 当前BlockCache的两种实现：LRUBlockCache和BucketCache，前者实现相对简单，后者在GC优化方面有明显提升
    - Region：数据表的一个分片，当表大小超过一定阈值会水平切分，分裂为两个Region。Region是集群负载均衡的基本单位。通常一张表的Region会分布在整个集群的多台RegionServer上，一个RegionServer上会管理多个Region，这些Region一般来自不同的数据表
        - 一个Region由一个或多个Store构成，Store的个数取决于表中列簇的个数。HBase中每个列簇的数据都集中存放在一起形成一个存储单元Store，**因此建议将具有相同IO特性的数据设置在同一个列簇中**
        - 每个Store由一个MemStore和一个或多个HFile组成。MemStore称为写缓存，用户写入数据时先写到MemStore，写满（默认128M）后系统会异步将数据flush成一个HFile文件。HFile文件数超过一定阈值之后系统会执行Compact操作进行小文件合并
#### 5. HDFS
- HBase底层依赖HDFS组件存储实际数据，包括用户数据文件、HLog日志文件等最终都会写入HDFS落盘。HBase内部封装了一个名为DFSClient的HDFS客户端组件，负责对HDFS的实际数据进行读写访问

### 1.4 HBase系统特性
#### 1. HBase的优点
- 容量巨大：可以支持单表千亿行、百万列的数据规模，数据容量可达TB甚至PB级别
- 良好的可扩展性：包括数据存储节点扩展以及读写服务节点扩展。存储节点可以通过增加HDFS的DataNode实现扩展，读写服务节点可以通过增加RegionServer节点实现计算层的扩展
- 稀疏性：允许大量列值为空，并不占用任何存储空间
- 高性能：擅长于OLTP场景，写操作性能强劲，对于随机单点读以及小范围的扫描读，其性能也能够得到保证。对于大范围的扫描读可以使用MapReduce提供的API以便实现更高效的并行扫描
- 多版本：一个KV可以同时保留多个版本，用户可以根据需要选择最新版本或某个历史版本
- 支持过期：支持TTL过期特性，超过TTL的数据会被自动清理
- Hadoop原生支持：用户可以直接绕过HBase系统操作HDFS文件，高效地完成数据扫描或者数据导入；也可以利用HDFS提供的多级存储特性将重要业务放到SSD，不重要业务放到HDD；或设置归档时间，进而将最近的数据放在SSD，将归档数据放在HDD。HBase对MapReduce的支持已有很多案例，后续还会针对Spark做更多工作
#### 2. HBase的缺点
- 不支持很复杂的聚合计算（如Join、GroupBy）等。如果业务需要使用，可以在HBase上架设Phoenix组件或Spark组件，前者主要应用于小规模聚合的OLTP场景，后者应用于大规模的OLAP场景
- 没有实现二级索引功能，所以不支持二级索引查找。但针对HBase的第三方二级索引方案非常丰富，比较普遍使用的是Phoenix提供的二级索引功能
- 原生不支持全局跨行事务，只支持单行事务模型。也可以使用Phoenix提供的全局事务模型组件来弥补
