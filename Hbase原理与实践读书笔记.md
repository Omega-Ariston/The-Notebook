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

## 第2章 基础数据结构与算法
### 2.1 跳跃表
- 跳跃表的插入、删除、查找操作的期望复杂度都是O(logN)
### 2.2 LSM树
- 与B+树相比，LSM树的索引对写入请求更友好，因为其会将写入操作处理为一次顺序写，而HDFS擅长的正是顺序写
- LSM树的索引一般由两部分组成：内存部分和磁盘部分。前者一般采用跳跃表来维护一个有序的KV集合，后者一般由多个内部KV有序的文件组成
- KeyValue存储格式：一般来说，LSM中存储的是多个KeyValue组成的集合，每个KeyValue一般都会用一个字节数组来表示，其主要分为以下几个字段：keyLen(4B), valueLen(4B), rowkeyLen(2B), rowkeyBytes, familyLen(1B), familyBytes, qualifierBytes（没有qualifierLen，因为其可以通过keyLen和其它字段的Len相减计算出来）, timestmap(8B), type(1B)
- LSM树的索引结构本质是将写入操作全部转化为磁盘的顺序写入，极大地提高了写入操作的性能。但是这种设计对读取操作非常不利，因为需要在读取的过程中通过归并所有文件来读取对应的KV，很消耗IO资源。因此HBase设计了异步的compaction来降低文件个数
- 对于HDFS而言，其只支持文件的顺序写，不支持文件的随机写，且HDFS擅长的场景是大文件存储而不是小文件，所以上层HBase选择LSM树作为索引结构是最合适的

### 2.3 布隆过滤器
- HBase1.x版本中的两种布隆过滤器：
    - ROW：按照rowkey来计算布隆过滤器的二进制串并存储。Get查询的时候必须带rowkey
    - ROWCOL：按照rowkey+family+qualifier这3个字段拼出byte[]来计算布隆过滤器值并存储，但在查询的时候，Get中只要缺少3个字段中任意一个，就无法通过布隆过滤器发觉他性能，因为key不确定
- 一般意义上的Scan都无法通过布隆过滤器来提升扫描数据性能，因为key不确定，但在某些特定场景下Scan操作可以借其提升性能，如对于ROWCOL类型的过滤器，在Scan操作中明确指定需要扫某些列。
- 在Scan过程中，碰到KV数据从一行换到新一行时，没法走ROWCOL布隆过滤器，因为新一行的Key值不确定，但是在同一行数据内切换列时可以进行优化，因为rowkey确定，同时column也已知

## 第4章 HBase客户端
### 4.1 HBase客户端实现
- HBase提供了面向Java、C/C++、Python等多种语言的客户端，非Java语言的客户端需要先访问ThriftServer，再通过其Java HBase客户端来请求HBase集群
#### 4.1.1 定位Meta表
- HBase一张表的数据由多个Region构成，而这些Region分布在整个集群的RegionServer上
- HBase系统内部设计了一张特殊的表——hbase:meta表，用于存放整个集群所有的Region信息
- HBase保证hbase:meta表始终只有一个Region，以确保其多次操作的原子性（HBase本质上只支持Region级别的事务）
- HBase客户端有一个叫做MetaCache的缓存，用于缓存业务rowkey所在的Region，这个Region可能有以下三种情况：
    1. Region信息为空，说明MetaCache中没有这个rowkey所在Region的任何Cache，需要去hbase:meta表中做Reversed Scan。首次查找时还需要向ZooKeeper请求hbase:meta表所在的RegionServer
    2. Region信息不为空，但是调用RPC请求对应RegionServer后发现Region并不在这个RegionServer上。说明MetaCache信息过期了，同样直接Reversed Scan后找到正确的Region并缓存
    3. Region信息不为空且调用RPC请求到对应Region
    Server后，发现是正确的RegionServer（绝大部分的请求都属于这种情况）
#### 4.1.2 Scan的复杂之处
- 用户每次执行scanner.next()，都会尝试去名为cache的队列中拿result。如果cache队列已经为空，则会发起一次RPC向服务端请求当前scanner的后续result数据。客户端收到result列表后，通过scanResultCache把这些results内的多个cell进行重组，最终组成用户需要的result放入到cache中。这个操作称为loadCache
- result重组的原因是RegionServer为了避免被当前RPC请求耗尽资源，实现了多个维度的资源限制，比如鼻梁骨out、单次RPC响应最大字节数等，一旦某个维度资源达到阈值，就马上把当前拿到的cell返回给客户端，因此客户端拿到的result可能不是一行完整的数据，因此需要对result进行重组
- Scan的几个重要概念：
    - caching：每次loadCache操作最多放caching个result到cache队列中，可以以此控制每次loadCache向服务端请求的数据量，避免出现音效scanner.next()操作耗时极长的情况
    - batch：用户拿到的result中最多含有一行数据中的batch个cell。如果某一行有5个cell，batch设置为2，那么用户会拿到3个result，它们包含的cell个数依次为2，2，1
    - allowPartial：用户能容忍拿到一行部分cell的result。设置此属性会跳过result中的cell重组，直接把服务端收到的result返回给用户
    - maxResultSize：loadCache时单次RPC操作最多拿到maxResultSize字节的结果集

### 4.2 HBase客户端避坑指南
- HBase中CAS接口的运行是Region级别串行执行的，其运行步骤为：
    1. 服务端拿到Region的行锁，避免出现两个线程同时修改一行数据，从而破坏行级别原子性的情况
    2. 等待该Region内的所有写入事务都已经成功提交并在mvcc上可见
    3. 通过Get操作拿到需要check的行数据，进行条件检查。若条件不符合，则终止CAS
    4. 将checkAndPut的put数据持久化
    5. 释放第1步拿到的行锁
- 在HBase2.x中对同一个Region内的不同行可以并行执行CAS，大大提高了Region内的CAS吞吐
- PrefixFilter前缀过滤器的实现很简单粗暴，Scan会一条条扫描，发现前缀不为指定值就读下一行，直到找到第一个rowkey前缀为该值的行为止。使用时可以简单加一个startRow，这样在Scan时会首先寻址到这个startRow，然后从这个位置扫描数据。最简单的方式是直接将PrefixFilter展开为startRow和stopRow，直接扫描区间数据
- PageFilter可以限定返回数据的行数以用于数据分页功能，但HBase里Filter状态全部都是Region内有效的，Scan一旦从一个Region切换到另一个Region，之前那个Filter的内部状态就无效了。这时用limit来实现更好
- HBase提供3种常见的数据写入API：
    1. table.put(put)：最常见的单行数据写入API，在服务端先写WAL，然后写MemStore，一旦MemStore写满就flush到磁盘上。默认每次写入都需要执行一次RPC和磁盘持久化。因此写入吞吐量受限于磁盘带宽、网络带宽以及flush的速度。但是它能保证每次写入操作都持久化到磁盘，不会有任何数据丢失。最重要的是能保证put操作的原子性
    2. table.put(List\<Put> puts)：在客户端缓存put，再打包通过一次RPC发送到服务端，一次性写WAL并写MemStore。可以省去多次往返RPC及多次刷盘的开销，吞吐量大大提升，但此RPC操作一般耗时会长一些，因为一次写入了多行数据。如果put分布在多个Region内，则不能保证这一批put的原子性，因为HBase并不提供跨Region的多行事务，其中失败的put会经历若干次重试
    3. bulk load：通过HBase提供的工具直接将待写入数据生成HFile，直接加载到对应Region下的CF内，是一种完全离线的快速写入方式，是最快的批量写手段。load完HFile之后，CF内部会进行Compaction（异步且可限速，IO压力可控），对线上集群非常友好
    