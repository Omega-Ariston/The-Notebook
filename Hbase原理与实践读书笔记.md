# HBase原理与实践
## 第1章 HBase概述
### 1.2 HBase数据模型
#### 1.2.1 逻辑视图
- HBase中的基本概念：
    - table：表，一个表包含多行数据
    - row：行，一行数据包含一个唯一标识rowkey、多个column以及对应的值。表中所有row都按照rowkey的字典序从小到大排序
    - column：列，与关系型数据库中的列不同，column由column family（列簇）以及qualifier（列名）两部分组成，中间使用“：”相连。column family在表他去的时候需要指定，用户不能随意增减。而一个column family下的qualifier可以设置任意多个，因此列可以动态增加，理论上可以扩展到上百万列
    - timestamp：时间戳，每个cell在写入HBase的时候都会默认分配一个时间戳作为该cell的版本，同一个rowkey、column下可以有多个value，版本号越大表示数据越新
    - cell：单元格，由五元组（row，column，timestamp，type，value）组成的结构，type表示put/delete这样的操作类型，timestamp表示版本。这个结构是以KV结构存储的，前四个元素组成的元组为key，value为value
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

## 第3章 HBase依赖服务
- 主要讲了Zookeeper和HDFS，暂时跳过

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
- result重组的原因是RegionServer为了避免被当前RPC请求耗尽资源，实现了多个维度的资源限制，比如timeout、单次RPC响应最大字节数等，一旦某个维度资源达到阈值，就马上把当前拿到的cell返回给客户端，因此客户端拿到的result可能不是一行完整的数据，因此需要对result进行重组
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

## 第5章 RegionServer的核心模块
- RegionServer组件实际上是一个综合体系，包含多个各司其职的核心模块：HLog、MemStore、HFile以及BlockCache
### 5.1 RegionServer内部结构
- 一个RegionServer由一个或多个HLog、一个BlockCache以及多个Region组成
- HLog用来保证数据写入的可靠性
- BlockCache可以将数据块缓存在内存中以提升数据读取性能
- Region是HBase中数据表的一个数据分片，一个RegionServer上通常会负责多个Region的数据读写。一个Region由多个Store组成，Store数量与列簇数量一致，每个Store存放对应列簇的数据，每个Store包含一个MemStore和多个HFile
### 5.2 HLog
- HLog一般只用于RegionServer宕机丢失数据时进行恢复或HBase主从复制
#### 5.2.1 HLog文件结构
- 每个RegionServer拥有一个或多个HLog，每个HLog是多个Region共享的
- HLog中，日志单元WALEntry表示一次行级更新的最小追加单元，由HLogKey和WALEdit两部分组成
- HLogKey由table name, region name以及sequenceid等字段构成
#### 5.2.2 HLog文件存储
- HBase中所有数据都存储在HDFS的指定目录
- HLog一般存储在/hbase/WALS和/hbase/oldWALs下，后者用于存储已过期的日志
- WALS目录下一般有多个子目录，每个子目录对应一个RegionServer域名+端口号+时间戳
#### 5.2.3 HLog生命周期
- HLog生命周期的4个阶段：
    1. HLog构建：HBase的任何写入操作都会先将记录追加写入到HLog文件中
    2. HLog滚动：HBase后台启动一个线程，每隔一段时间进行日志滚动。会新建一个新的日志文件，接收新的日志数据，为了方便过期日志数据能够以文件的形式直接删除
    3. HLog失效：写入数据一旦从MemStore中落盘，对应的日志数据就会失效。因此查看日志是否失效只需查看该日志对应的数据是否已经完成落盘。失效的日志文件会移动到oldWALs文件夹（此时还未被删除）
    4. HLog删除：Master后台会启动线程每隔一段时间检查一次oldWALs下所有失效日志文件，确认是否可以删除并相应地执行删除操作。确认条件有两个：该HLog文件是否还在参与主从复制；该HLog文件是否已经在oldWALs目录中存在TTL时间（默认10分钟）
### 5.3 MemStore
- HBase中一张表会被水平切分成多个Region，每个Region负责自己区域的数据读写请求。水平切分意味着每个Region都有所有列簇数据，不同列簇数据由不同Store存储，每个Store都有一个MemStore和一系列HFile
- 由于HDFS本身只允许顺序读写，不能更新，因此数据在落盘前生成HFile之前需要完成排序工作，MemStore就是KV数据排序的实际执行者
- MemStore也作为一个缓存级的存储组件，缓存着最近写入的数据
#### 5.3.1 MemStore内部结构
- HBase使用了JDK自带的ConcurrentSkipListMap实现MemStore，其底层使用跳跃表来保证数据的有序性和各操作的O(logN)时间复杂度，并采用CAS原子性操作保证线程安全（避免锁开销）
- MemStore由两个ConcurrentSkipListMap实现，写入操作会将数据写入Map A，当其数据量超过一定阈值后会创建一个Map B来接收用户新的请求，之前已经写满的Map A会执行异步flush操作落盘形成HFile
#### 5.3.2 MemStore的GC问题
- MemStore的工作模式会容易引起严重的内存碎片，因为一个RegionServer由多个Region构成，每个Region根据列簇的不同又包含多个MemStore，这些MemStore都是共享内存的。对于JVM而言所有MemStore的数据都是混合在一起写入Heap，此时如果某个Region上对应的所有MemStore执行落盘操作，则JVM会产生内存条带
- 用于改善上述情况的方案：MSLAB内存管理方式、MemStore Chunk Pool。这两种方案日后可以深入研究下
### 5.4 HFile
#### 5.4.1 HFile逻辑结构
- HFile文件分为四个部分：
1. Scanned Block：表示顺序扫描HFile时所有数据块交付被读取。这个部分包含Data Block, Leaf Index Block以及Bloom Block。Data Block存储用户的KV数据，Leaf Index Block存储索引树的叶子节点，Bloom Block存储布隆过滤器相关数据
2. Non-scanned Block：表示在HFile顺序扫描的时候数据不会被读取，主要包括Meta Block和Intermediate Level Data Index Blocks两部分
3. Load-on-open：这部分数据会在RegionServer打开HFile时直接加载到内存中，包括FileInfo、布隆过滤器MetaBlock、Root Data Index和Meta IndexBlock
4. Trailer：主要记录了HFile的版本信息、其它各个部分的偏移值和寻址信息
#### 5.4.2 HFile物理结构
- HFile由不同类型的Block构成，虽然它们类型不同，但数据结构相同
- Block大小可以在创建表列簇的时候指定，默认64K。通常来说，大号的Block有利于大规模顺序扫描，小号的Block更有利于随机查询
- HFileBlock支持两种类型，一种含有checksum，一种不含
- HFileBlock主要包含两个部分：BlockHeader和BlockData。其中Header主要存储Block相关元数据，Data用来存储具体数据
- BlockHeader中最核心的字段是BlockType，表示该Block的类型
- HBase中最核心的几种BlockType：
    - Trailer Block：记录HFile基本信息，文件中各个部分的偏移量和寻址信息
    - Meta Block：存储布隆过滤器相关元数据信息
    - Data Block：存储用户KV信息
    - Root Index：HFile索引树根索引
    - Intermediate Level Index：HFile索引树中间层级索引
    - Leaf Level Index：HFile索引树叶子索引
    - Bloom Meta Block：存储Bloom相关元数据
    - Bloom Block：存储Bloom相关数据
#### 5.4.3 HFile的基础Block
1. Trailer Block
    - RegionServer在打开HFile时会加载并解析所有HFile的Trailer部分，再根据元数据内容进一步加载load-on-open部分的数据（需要知道对应的偏移量和数据大小）
2. Data Block
    - Data Block是HBase中文件读取的最小单元。主要存储用户的KV数据
    - KV数据由4个部分组成：KeyLen、ValueLen、Key、Value，其中两个Len的长度是固定的
#### 5.4.4 HFile中与布隆过滤器相关的Block
- HBase会为每个HFile分配对应的位数组，用于存储hash函数的映射信息。
- 当HFile文件越大，里面存储的KV值越多，位数组就会相应越大，不利于直接加载进内存。因此HFile V2将位数组进行了拆分，一部分连续的Key使用一个位数组。当根据Key进行查询时先定位到具体的位数组，再进行过滤
- 文件结构上每个位数组对应一个Bloom Block（位于scanned block部分），为了方便Key定位到对应的位数组，HFile V2设计了相应的Bloom Index Block（位于load-on-open部分）
#### 5.4.5 HFile中索引相关的Block
- HFile V2中引入了多级索引，用于解决索引数据太大无法全部加载到内存中的问题
- HFile中索引是树状结构，Root Index Block位于load-on-open部分，Intermediate Index Block位于Non-Scanned Block部分，Leaf Index Block位于scanned block部分
- 刚开始Data Block数据量小时Root Index Block会直接指向Data Block。当Root Index Block大小超过阈值后索引会分裂为多级结构，变为两层：根->叶。数据量再变大则会变为三层：根->中间->叶
1. Root Index Block
    - IndexEntry表示具体的索引对象，每个索引对象由3个字段组成：Block Offset, BlockDataSize, BlockKey（索引指向Data Block中的第一个Key）
    - 除此之外还有3个字段用来记录MidKey的相关信息，用于在对HFile进行split操作时，快速定位HFile的切分点位置
2. NonRoot Index Block
    - 在Root Index Block的3个核心字段基础上增加了内部索引Entry Offset字段，用于记录该Index Entry在Block中的相对偏移量，以实现Block内的二分查找。
    - 所有非根节点索引块在内部定位一个Key的具体索引都不是通过遍历实现，而是使用二分查找，效率更高
#### 5.4.7 HFile V3版本
- V3版本新增了对cell标签功能的支持，为其它与安全相关的功能，如单元级ACL和单元级可见性提供了实现框架

### 5.5 BlockCache
- BlockCache是RegionServer级别的，一个RegionServer只有一个BlockCache，在其启动时就要完成初始化工作
- 当前的三种BlockCache实现方案：LRUBlockCache、SlabCache、BucketCache。三种方案不同之处主要在于内存管理模式，其中LRU为JVM堆内方案，后两者则允许将部分数据存储在堆外
#### 5.5.1 LRUBlockCache
- HBase默认的BlockCache机制，用ConcurrentHashMap管理BlockKey到Block的映射关系，使用LRU淘汰算法将最近最少使用的Block置换出去
1. 缓存分层策略
    - HBase将整个BlockCache分为三个部分：single-access, multi-access和in-memory，分别占25%, 50%, 25%
    - 在一次随机读中，一个Block从HDFS加载出来之后首先放入single-access区，后续如果有多次请求访问到这个Block，会将这个Block移到multi-access区，而in-memory区数据常驻内存，一般用来存储元数据这类访问频繁且量小的数据
    - 建表的时候可以设置列簇属性IN_MEMORY=TRUE，该列簇的Block在从磁盘中加载出来后会直接放入in-memory区，但要注意这个区里一般都是系统元数据表，比如hbase:meta和hbase:namespace，所以放东西进去时要小心别把它们挤出去了，免得影响所有业务
2. LRU淘汰算法实现
    - 在每次cache block时系统将BlockKey和Block放入HashMap后都会检查BlockCache总量是否达到阈值，如果达到，则唤醒淘汰线程对Map中的Block进行淘汰
    - 系统设置了3个MinMaxPriorityQueue，用于对3个缓存层分别进行LRU淘汰
3. LRUBlockCache方案缺点
    - 数据对应的内存对象在JVM中随着时间流逝会进入老年代，老年代的Block被淘汰后会变为内存垃圾，最终由CMS回收，带来大量内存碎片

#### 5.5.2 SlabCache
- 使用Java NIO DirectByteBuffer技术实现堆外内存存储，以解决上一个方案中的GC问题
- 默认在系统初始化时分配两个缓冲区，分别占BlockCache大小的80%和20%
- 两个缓冲区分别存储小于等于64K和小于等于128K的Block，所以Block太大时两个缓冲区都无法缓存
- 当用户设置的Block大小大于128K时，SlabCache方案会失效，此时一般要搭配LRUCache做DoubleBlockCache。Block从HDFS加载出来后会在两个Cache中分别存储一份，读时首先在LRU中找，miss了再去Slab中找，如果命中，就将该Block放入LRU
- SlabCache固定大小内存设置会导致实际内存使用率较低，且搭配LRU使用仍然会导致GC问题，HBase0.98版本后已不建议使用此方案

#### 5.5.3 BucketCache
- BucketCache有有一种工作模式：heap、offheap和file，分别使用JVM、DirectByteBuffer技术和类似SSD的存储介质来缓存Data Block
- BucketCache会申请许多带有固定大小标签的Bucket，但与SlabCache不同，BucketCache会在初始化时申请14种不同大小的Bucket，当某一种Bucket空间不足时，系统会从其它Bucket空间借用内存，以避免内存使用率低的情况
- 实际实现中，HBase将BucketCache和LRUCache搭配成为CombinedBlockCache，LRU用于存储Index Block和Bloom Block，而将Data Block存储在BucketCache中
- 一次随机读需要先从LRUCache中查到对应的Index Block，再到BucketCache中查找对应Data Block
- BucketCache极大降低了LRU的内存碎片问题，但使用堆外内存会存在拷贝内存的问题，一定程度上影响读写性能（2.0版本中得到了解决）
1. BucketCache的内存组织形式
    - HBase使用BucketAllocator类实现对Bucket的组织管理
    - HBase会根据每个Bucket的size标签对Bucket进行分类，相同size的Bucket由同一个BucketSizeInfo管理。
    - BucketSize总会比Block本身大1KB，因为Block本身并不严格固定大小，总会大那么一点
    - HBase启动时就决定了size标签的分类，默认从4+1K到512+1K
    - Bucket的size标签可以动态调整，比如空闲的Bucket可以转换为空间不足的Bucket，但会至少保留一个该size的Bucket
2. BucketCache中Block缓存写入、读取流程
    - BucketCache的5个模块：
        1. RAMCache：是一个存储blockKey和Block对应关系的HashMap
        2. WriteThread：整个Block写入的中心枢纽，负责异步将Block写入到内存空间
        3. BucketAllocator：实现对Bucket的组织管理，为Block分配内存空间
        4. IOEngine：具体的内存管理模块，将Block数据写入对应地址的内存空间
        5. BackingMap：用于存储blockKey与对应物理内存偏移量的映射关系的HashMap，用于根据blockKey定位具体的Block
    - Bucket缓存的写入流程：
        1. 将Block写入RAMCache。实际实现中HBase设置了多个RAMCache，系统根据blockKey进行hash，根据hash结果将Block分配到对应RAMCache
        2. WriteThread从RAMCache中取出所有Block。HBase会同时启动多个WriteThread并发地执行异步写入，每个WriteThread对应一个RAMCache
        3. 每个WriteThread会遍历RAMCache中所有Block，分别调用bucketAllocator为这些Block分配内存空间
        4. BucketAllocator选择与Block大小对应的Bucket进行存放并返回对应的物理地址偏移量offset
        5. WriteThread将Block以及分配好的物理地址偏移量传给IOEngine模块，执行具体的内存写入操作
        6. 写入成功后，将blockKey与对应物理内存偏移量的映射关系写入BackingMap中，用于后续查找
    - Bucket缓存的读取流程：
        1. 首先从RAMCache中查找，还没来得及写入Bucket的Block会在这
        2. 如果在RAMCache中没有，再根据blockKey在BackingMap中找到对应的物理偏移地址量offset
        3. 根据物理偏移地址offset直接从内存中查找对应的Block数据
3. BucketCache工作模式
    - BucketCache三种工作模式在内存逻辑组织形式以及缓存流程上都是相同的，但因三者对应的最终存储介质有所不同，IOEngine也会有所不同
    - heap和offheap都是使用内存作为存储介质，内存分配查询使用Java NIO ByteBuffer技术，前者使用ByteBuffer.allocate()，后者使用ByteBuffer.allocateDirect()
    - offheap模式需要将缓存先从操作系统拷贝到JVM heap，会更费时

## 第6章 HBase读写流程
### 6.1 HBase写入流程
- HBase采用LSM树架构，适用于写多读少的应用场景
- HBase服务端并没有提供update和delete接口，其对数据的更新、删除操作在服务器端也认为是写入操作，更新操作会写入一个最新版本数据，删除操作会写入一条标记为deleted的KV数据，因此流程与写入流程完全一致
#### 6.1.1 写入流程的三个阶段
- 从整体架构视角来看，写入流程可以概括为三个阶段：
    1. 客户端处理阶段：客户端将用户的写入请求进行预处理，根据集群元数据定位到数据所在的RegionServer，将请求发送给对应的RegionServer
    2. Region写入阶段：RegionServer接收到写入请求之后将数据解析出来，首先写入WAL，再写入对应Region列簇的MemStore
    3. MemStore Flush阶段：当Region中MemStore容量超过一定阈值，系统会异步执行flush操作，将内存中的数据写入文件，形成HFile
- 注意：用户写入请求在完成Region MemStore写入后会返回成功，因为后续的Flush是一个异步过程
- 第1步的RPC请求会经过Protobuf序列化后发送给对应的RegionServer
- 第2步在执行完检查（如Region是否只读、MemStore大小是否超过blockingMemStoreSize等）后会执行一系列核心操作，可以简单概括为获取行锁->更新Timestamp->为单次写入构建WALEdit->将WALEdit写入HLog->将数据写入MemStore->释放行锁->HLog同步到HDFS->结束写事务并对读请求可见
#### 6.1.2 Region写入流程
- HLog的持久化等级：
    1. SKIP_WAL：只写缓存，不写HLog日志，性能佳，但数据有丢失风险
    2. ASYNC_WAL：异步将数据写入HLog日志中
    3. SYNC_WAL：同步将数据写入日志文件中，但只保证写入文件系统，不一定有落盘
    4. FSYNC_WAL：同步将数据写入日志文件并强制落盘，可保证数据不会丢失，但性能较差
    5. USER_DEFAULT：默认HBase使用SYNC_WAL等级
- HLog的写入模型：
    - HLog写入需要经过三个阶段：写本地缓存->写文件系统->写磁盘
    - 三个阶段可以流水线工作，并使用生产者-消费者模型
    - HBase使用LMAX Disruptor框架实现了无锁有界队列操作
- 随机写入MemStore：
    - KV写入MemStore并不会每次都随机在堆上创建一个内存对象，而是会通过MemStore-Local Allocation Buffer（MSLAB）机制预先申请一个大的（2M）Chunk内存，写入的KV会进行一次封装，顺序拷贝到这个Chunk中，以避免出现JVM内存碎片
    - MemStore的3步写入流程：
        1. 检查当前可用的Chunk是否写满，若满，则重新申请一个2M的Chunk
        2. 将当前KV在内存中重新构建，在可用Chunk的指定offset处申请内存创建一个新的KV对象
        3. 将新创建的KV对象写入ConcurrentSkipListMap中
#### 6.1.3 MemStore Flush
1. 触发条件
    - MemStore级别限制：当Region中任意一个MemStore大小达到指定的上限（默认128M）
    - Region级别限制：当Region中所有MemStore大小总和达到上限
    - RegionServer级别限制：当RegionServer中MemStore的大小总和超过低水位阈值，RegionServer开始强制执行flush，从MemStore最大的Region开始依次flush，直到总MemStore大小下降到低水位阈值
    - 当RegionServer中HLog数量达到上限，会选取最早的HLog对应的一个或多个Region进行flush
    - HBase定期刷新MemStore：默认周期为1小时，有一定时间的随机延时
    - 手动执行flush，通过shell命令flush 'tablename'或flush 'regionname'
2. 执行流程
    - HBase采用类似两阶段提交的方式将flush过程分为三个阶段，以减少flush过程对读写的影响：
        1. prepare：遍历当前Region中所有MemStore，将当前数据集做一个snapshot，再新建一个数据集接收新的数据写入，此阶段需要添加updateLock对写请求阻塞，结束后释放（持锁很短）
        2. flush：遍历所有MemStore，将上一阶段生成的snapshot持久化为临时文件统一放在./tmp目录下。因为涉及磁盘IO，这个过程比较慢
        3. commit：遍历所有MemStore，将上一阶段生成的临时文件移到指定的ColumnFamily目录下，针对HFile生成对应的Storefile和Reader，把storefile添加到Store的storefiles列表中，再清空prepare阶段生成的snapshot
3. 生成Hfile
    - KV数据生成HFile，首先会构建Bloom Block以及Data Block，一旦写满一个Data Block就会将其落盘同时构造一个Leaf Index Entry，写入Leaf Index Block，直到Leaf Index Block写满落盘
    - 每写入一个KV都会动态地去构建scanned block部分，等所有的KV都写入完成后再静态地构建non-scanned block、load-on-open以及trailer部分
- 注意：大部分MemStore Flush操作不会对业务读写产生太大影响很大，比如定期刷新、手动执行、MemStore级别限制、HLog数量限制、Region级别限制，只会阻塞对应Region上的写请求，且阻塞时间较短。然而一旦触发RegionServer级别限制导致flush，会对用户请求产生较大的影响，会阻塞所有落在该RegionServer上的写入操作

### 6.2 BulkLoad功能
- Bulkload功能使用场景：用户数据位于HDFS中，业务需要定期将这部分海量数据导入HBase系统，以执行随机查询更新操作
- Bulkload原理：首先使用MapReduce将待写入数据转换为HFile文件，再将这些HFile文件加载到在线集群，无需将写请求发给RegionServer处理
#### 6.2.1 BulkLoad核心流程
1. HFile生成阶段
    - 这个阶段会运行一个MR任务，mapper需要自己实现，将HDFS文件中的数据读出来组装成一个复合KV，其中key是rowkey，value可以是KV对象、put对象或delete对象；reducer由HBase负责
    - reducer的工作事项：
        - 根据表信息配置一个全局有序的partitioner
        - 将partitioner文件上传到HDFS集群并写入分布式缓存
        - 将task个数设置为目标表Region的个数
        - 设置输出key/value类，使其满足HFileOutputFormat格式要求
        - 设置reducer执行相应排序
2. HFile导入阶段
    - 使用工具completebulkload将HFile加载到在线HBase集群
    - completebuload的工作事项：
        - 依次检查所有HFile文件，将每个文件映射到对应的Region
        - 将HFile文件移动到对应Region所在的HDFS目录下
        - 告知Region对应的RegionServer，加载HFile文件对外提供服务
- Bulkload的数据源一定在HDFS上，如果需要通过Bulkload将MySQL中的数据导入HBase，则需要先将数据转换成HDFS上的文件
- 当用户生成的HFile所在的HDFS集群和HBase所在的HDFS集群是同一个时，能保证HFile与目标Region落在同一个机器上（可由参数开关）。当不是同一个集群时，会无法保证locality，需要在跑完Bulkload后手动执行major compact
- HBase1.3之前的版本中，Bulkload的数据不会被复制到peer集群，可能导致peer集群少数据。1.3之后可以通过参数开启Bulkload数据复制
### 6.3 HBase读取流程
- HBase读流程比写流程复杂得多，主要基于两方面原因：一是因为HBase一次范围查询可能涉及多个Region、多块缓存甚至多个数据存储文件；二是因为HBase中更新及删除操作实现简单，并没有更新和删除原有数据，真正的数据删除发生在major compact时。因此数据读取需要根据版本进行过滤，并对已经标记删除的数据进行过滤
- HBase读取流程可分为四个步骤：
    1. Client-Server读取交互逻辑
    2. Server端Scan构架体系
    3. 过滤淘汰不符合查询条件的HFile
    4. 从HFile中读取待查找Key
#### 6.3.1 Client-Server读取交互逻辑
- HBase数据读取可分为get和scan两类，但其实可统一为scan一类，因为get其实就是scan了一行数据
- Client-Server端的Scan操作并没有设计为一次RPC请求，因为扫描结果可能会很大，HBase会根据设置条件将一次大scan操作拆分为多个RPC请求，每个RPC请求称为一次next请求，只返回规定数量的结果，前面的章节中已有相应描述
#### 6.3.2 Server端Scan框架体系
- HBase中每个Region都是一个独立的存储引擎，客户端可以将每个子区间请求分别发送给对应的Region进行处理
- RegionServer接收到客户端的get/scan请求后做了两件事情：首先构建scanner iterator体系；然后执行next函数获取KV，并对其进行条件过滤
- Scanner的核心体系包括三层Scanner：Region Scanner、StoreScanner、MemStoreScanner+StoreFileScanner
- 一个RegionScanner由多个StoreScanner构成，StoreScanner数量与列簇数量一致，并一一对应负责Store的数据查找
- 一个StoreScanner由MemStoreScanner和StoreFileScanner构成，每个Store的数据由内存中的MemStore和磁盘上的StoreFile文件组成
- 每个HFile都会由StoreScanner构造出一个StoreFileScanner，用于执行对应文件的检索，MemStore也一样
- RegionScanner和StoreScanner并不负责实际查找操作，更多地承担组织调度的任务
- KeyValueScanner会将该Store中所有StoreFileScanner和MemStoreScanner合并形成一个最小堆，并通过不断地pop来完成归并排序
- 条件过滤会在KeyValueScanner执行next函数获取KV时进行
#### 6.3.3 过滤淘汰不符合查询条件的HFile
- 主要有有三种过滤手段：根据KeyRange、根据TimeRange、根据布隆过滤器
#### 6.3.4 从HFile中读取待查找Key
- 在一个HFile文件中seek待查找的key，可以分解为4步
1. 根据HFile索引树定位目标Block
    - 多级索引中只有根节点常驻内存，其余需要涉及IO查询，但Block有缓存机制可以提升性能
2. BlockCache中检索目标Block
    - 通过前文中提到的BackingMap
3. HDFS文件中检索目标Block
    - 读取HFile的命令会下发到HDFS
    - NameNode做的两件事情：找到属于这个HFile的所有HDFSBlock列表，并确认数据在其中哪个Block上；找到Block后定位该Block存在于哪些DataNode上，选择一个最优的返回给客户端
4. 从Block中读取待查找KV
    - HFile Block由KV大小到大排序构成，但这些KV并不固定长度，因此只能遍历扫描查找
### 6.4 深入理解Coprocessor
- HBase的Coprocessor机制使用户可以将自己编写的代码运行在RegionServer上，大多数情况下用户用不上这个功能
- Coprocessor分为两种：
    1. Observer
        - 类似于MySQL中的触发器，提供钩子使用户代码在特定事件发生之前或之后得到执行，比如在执行put或者get操作之前检查用户权限
        - 当前HBase系统中提供4种Observer接口
        1. RegionObserver：主要监听Region相关事件，比如get, put, scan, delete以及flush等
        2. RegionServerObserver：主要监听RegionServer相关事件，比如RegionServer启动、关闭，或者执行Region合并等事件
        3. WALObserver：主要监听WAL相关事件，比如WAL写入、滚动等
        4. MasterObserver：主要监听Master相关事件，比如建表、删表以及修改表结构等
    2. Endpoint
        - 类似于MySQL中的存储过程，允许将用户代码下推到数据层执行
        - 用户可以自定义一个客户端与RegionServer通信的RPC调用协议，通过RPC调用执行部署在服务器端的业务代码
- Observer的钩子函数执行对用户透明，无需显式调用，而Endpoint的执行必须由用户显式触发调用
- Coprocessor可以通过配置文件静态加载（需要重启集群）或动态加载（使用shell或HTableDescriptor中的相应函数）

## 第7章 Compaction实现
- 一般基于LSM树体系架构的系统都会设计Compaction，比如LevelDB、RocksDB以及Cassandra等
### 7.1 Compaction工作原理
- Compaction是从一个Region的一个Store中选择部分HFile文件进行合并，先从这些待合并的数据文件中依次读出KV，再由小到大排序后写入一个新的文件，之后这个新文件会取代之前已合并的所有文件对外提供服务
- HBase根据合并规模将Compaction分为两类：
    1. Minor Compaction：选取部分小的、相邻的HFile，将它们合并成一个更大的HFile
    2. Major Compaction：将一个Store中所有的HFile合并成一个HFile，这个过程中还会完全清理三类无意义数据：被删除的数据、TTL过期数据、版本号超过设定版本号的数据
- 一般情况下Major Compaction持续时间比较长，消耗大量系统资源，对上层业务有比较大的影响，一般推荐关闭自动触发Major Compaction，在业务低峰期手动触发
- HBase中Compaction的核心作用：
    - 合并小文件，减少文件数，稳定随机读延迟
    - 提高数据的本地化率
    - 清除无效数据，减少数据存储量
- Compaction合并小文件时会将落在远程Datanode上的数据读取出来重新写入大文件，合并后的大文件在当前DataNode节点上有一个副本，因此提高了数据的本地化率
- Compaction过程中的明显副作用：将小文件的数据读出来需要IO，很多小文件数据跨网络传输需要带宽，读出来之后写入大文件，因为是三副本写入，所以需要网络及IO开销
- Compaction的本质是使用短时间的IO消耗以及带宽消耗换取后续查询的低延迟
#### 7.1.1 Compaction基本流程
- Compaction会被交由一个独立的线程处理，该线程首先会从对应Store中选择合适的HFile文件进行合并（整个Compaction的核心）
- 选择文件时最理想的情况是选取IO负载重、文件小的文件集，实际实现中HBase提供了多种文件选取算法，如RatioBasedCompactionPolicy、ExploringCompactionPolicy、和StripeCompactionPolicy等，用户也可以通过实现接口实现自己的Compaction策略
- 选出文件后HBase会根据这些HFile文件总大小挑选对应的线程池处理，最后对这些文件执行具体的合并操作
#### 7.1.2 Compaction触发时机
- 最常见的有三种触发时机
1. MemStore Flush：每次MemStore flush完都会检查当前Store中的文件数，一旦超过阈值就会触发Compaction。Compaction以Store为单位，而在flush触发条件下整个Region的所有Store都会执行compact检查，**所以一个Region有可能在短时间内执行多次Compaction**
2. 后台线程周期性检查：RegionServer会在后台启动一个线程CompactionChecker，定期触发检查对应Store是否需要执行Compaction，该线程优先检查Store中总文件数是否大于阈值，一旦大于就触发Compaction；如果不满足，接着检查是否满足Major Compaction条件（当前Store中HFile最早更新时间是否早于某个值mcTime，一般是7天），此检查可以被禁用
3. 手动触发：大多是为了执行Major Compaction，原因通常有三个：
    1. 业务担心自动Major Compaction影响性能，选择低峰期手动触发
    2. 用户执行完alter后希望立即生效
    3. HBase管理员发现硬盘容量不够时，删除大量过期数据
#### 7.1.3 待合并HFile集合选择策略
- HBase早期版本的两种策略：RatioBased和Exploring
- 两者都会首先对该Store中所有HFile逐一进行排查，排除当前正在执行Compaction的文件以及比这些文件更新的所有文件、某些过大的文件（默认Long.MAX_VALUE），以免产生大量IO消耗
- 排除后留下来的文件称为候选文件，接下来HBase判断候选文件是否满足Major Compaction的条件，只要满足其中一条即可：
    - 用户强制执行Major Compaction
    - 长时间（上次执行的时间早于当前时间减hbase.hregion.majorcompaction值）没有进行Major Compaction且候选文件数小于指定值（默认10）
    - Store中含有reference文件（region分裂产生的临时文件）
- 如果满足Major Compaction条件，则文件选择直接结束，因为所有文件都要参加合并
- 如果不满足Major Compaction条件，则为Minor Compaction，使用RatioBased或Exploring进行选择：
1. RatioBasedCompactionPolicy
    - 从老到新逐一扫描所有候选文件，满足其中一个条件则停止扫描
    1. 当前文件大小<比当前文件新的所有文件大小总和*ratio，ratio为可变值，高峰期为1.2，非高峰期为5（非高峰期允许compact更大的文件），可以通过参数设置高峰期时间段
    2. 当前所剩候选文件数<=指定值（默认3）
    - 停止扫描后，待合并的文件就是当前扫描文件和比它更新的所有文件
2. ExploringCompactionPolicy
    - 与Ratio策略找到一个合适文件后就停止扫描不同，Exploring会记录所有合适的文件集合，并在这些文件集合中寻找最优解
    - 最优解：待合并文件数最多或待合并文件数相同的情况下文件较小的
#### 7.1.4 挑选合适的执行线程池
- HBase中有一个专门的类CompactSplitThread负责接收Compaction请求和split请求，其内部构造了多个线程池：splits线程池负责处理所有split请求，largeCompactions用来处理大Compaction，smallCompaction负责处理小compaction
- 上述设计的目的是能够将请求独立处理，提高系统的处理性能
- 大Compaction并不是Major Compaction，小Compaction也并不是Minor Compaction，而是根据设置的阈值而定
- largeCompactions和smallCompactions线程池默认都只有一个线程，可以通过参数进行调整

#### 7.1.5 HFile文件合并执行
- 合并流程主要分为以下几步：
    1. 分别读出待合并HFile文件的KV，进行归并排序处理，写到./tmp目录下的临时文件中
    2. 将临时文件移动到对应Store的数据目录
    3. 将Compaction的输入文件路径和输出文件路径封装为KV写入HLog日志，并打上Compaction标记，最后强制执行sync
    4. 将对应Store数据目录下的Compaction输入文件全部删除
- 如果RegionServer在步骤2之前发生异常，本次Compaction会被认定为失败，下次可以重头来过，唯一的影响就是多了一份多余的数据
- 如果RegionServer在步骤2和3之间发生异常，也仅会多一份冗余数据
- 如果在步骤3和4之间发生异常，RegionServer在重新打开Region后首先会从HLog中看到标有Compaction的日志，因为此时输入文件和输出文件已经持久化到HDFS，只需要根据HLog移除Compaction输入文件即可

#### 7.1.6 Compaction相关注意事项
- Compaction执行阶段的读写吞吐量需要进行限制，以防止短时间大量系统资源消耗导致的用户业务读写延迟抖动
1. Limit Compaction Speed
    - 正常情况下用户需要设置吞吐量下限参数和上限参数
    - 如果当前Store中File数量太多，且超过了blockingFileCount，此时所有写请求会被阻塞以等待Compaction的完成，此时上述限制会自动失效
2. Compaction BandWith Limit
    - 与前者思路基本一致，主要涉及两个参数
    - compactBwLimit：一次Compaction的最大带宽使用量，实际使用带宽高于该值时，会强制其sleep一段时间
    - numOfFilesDisableCompactLimit：写请求非常大的情况下，限制带宽使用会导致HFile规程，影响读请求响应延时，因此一旦Store中HFile数量超过该值，带宽限制就会失效

### 7.2 Compaction高级策略
- 优化Compaction的共性特征：
    1. 减少参与Compaction的文件数，尽量不要合并大文件
    2. 不要合并不需要合并的文件，如OpenTSDB应用场景下的老数据，基本不会被查询，合不合不影响性能
    3. 小Region更有利于Compaction，因为大Region会生成大量文件，小Region只会生成少量文件，不会引起显著的IO放大
- 这一部分还介绍了一些别出心裁的Compaction策略，值得日后回过头来研究

## 第8章 负载均衡实现
### 8.1 Region迁移
- 实际执行分片迁移的两个步骤：先根据负载均衡策略制定分片迁移计划，再根据迁移计划执行分片的实际迁移
1. Region迁移的流程
    - Region迁移虽然轻量级，但实现逻辑比较复杂，主要体现在两个方面：迁移过程中涉及多种状态的改变；迁移过程中涉及Master、ZooKeeper以及RegionServer等多个组件的相互协调
    - 实际过程中，Region迁移分为两个阶段：unassign阶段和assign阶段
    - unassign阶段：表示Region从源RegionServer上下线
        1. Master生成事件M_ZK_REGION_CLOSING并更新到ZooKeeper组件，同时将本地内存中该Region的状态修改为PENDING_CLOSE
        2. Master通过RPC发送close命令给拥有该Region的RegionServer，令其关闭该Region
        3. RegionServer接收到Master发送过来的命令后，生成一个RS_ZK_REGION_CLOSING事件，更新到ZooKeeper
        4. Master监听到ZooKeeper节点变动后，更新内存中Region的状态为CLOSING
        5. RegionServer执行Region关闭操作。如果该Region正在执行flush或者Compaction，等待操作完成；否则将该Region下的所有MemStore强制flush，然后关闭Region相关的服务
        6. 关闭完成后生成事件RS_ZK_REGION_CLOSED，更新到ZooKeeper。Master监听到ZooKeeper节点变动后，更新该Region状态为CLOSED
    - assign阶段：表示Region在目标RegionServer上上线
        1. Master生成事件M_ZK_REGION_OFFLINE并更新到ZooKeeper组件，同时将本地内存中该Region的状态修改为PENDING_OPEN
        2. Master通过RPC发送open命令给拥有该Region的RegionServer，令其打开该Region
        3. RegionServer接收到Master发送过来的命令后，生成一个RS_ZK_REGION_OPENING事件，更新到ZooKeeper
        4. Master监听到ZooKeeper节点变动后，更新内存中Region的状态为OPENING
        5. RegionServer执行Region打开操作，初始化相应的服务
        6. 打开完成后生成事件RS_ZK_REGION_OPENED，更新到ZooKeeper，Master监听到ZooKeeper节点变动后，更新该Region的状态为OPEN
    - 总体来看，整个过程涉及Master、RegionServer和ZooKeeper三个组件，它们的主要职责如下：
        - Master负责维护Region在整个操作过程中的状态变化，起到枢纽的作用
        - RegionServer负责接收Master的指令执行具体的unassign/assign操作，实际上就是关闭Region或者打开Region操作
        - ZooKeeper负责存储操作过程中的事件。ZooKeeper有一个路径为/hbase/region-in-transition的节点，一旦Region发生unassign操作，就会在这个节点下生成一个子节点，内容是“事件”经过序列化的字符串，并且Master会在这个子节点上监听
2. Region In Transition
    - 迁移操作为什么需要设置这些状态？因为unassign和assign都是由多个子操作组成，涉及多个组件的协调合作，需要记录状态用以跟踪进度，以便于在异常发生后根据进度继续执行
    - 管理状态的方式：
    1. meta表：只存储Region所在的RegionServer
    2. Master内存：存储整个集群所有的Region信息，状态变更由RegionServer通知到ZooKeeper再通知到Master，所以会有信息滞后。在HBase Master WebUI上看到的Region状态均来自于此
    3. ZooKeeper的region-in-transition节点：存储临时性状态转移信息，作为Master和RegionServer间反馈Region状态的通道
    - 当这三个状态不一致时，就会出现Region In Transition（RIT）现象
    - Region在迁移的过程中必然会出现短暂的RIT状态，无需任何人工干预操作
### 8.2 Region合并
- 用得不多，一个典型的应用场景：在某些业务中本来接收写入的Region在之后的很长时间都不再接收任何写入，而且Region上的数据因为TTL过期被删除，这种场景下的Region实际上没有任何存在意义，称为空闲Region。空闲Region过多会导致集群管理运维成本增加，可以使用在线合并功能将这些Region与相邻Region合并
- 从原理上看，Region合并的主要流程如下：
    1. 客户端发送merge请求给Master
    2. Master将待合并的所有Region都move到同一个RegionServer上
    3. Master发送merge请求给该RegionServer
    4. RegionServer启动一个本地事务执行merge操作
    5. merge操作将待合并的两个Region下线，并将两个Region的文件进行合并
    6. 将这两个Region从hbase:meta中删除，并将新生成的Region添加到hbase:meta中
    7. 将新生成的Region上线
- HBase使用merge_region命令进行Region合并，其是一个异步操作，需要用户在一段时间后手动检测合并是否成功
- 默认情况下merge_region命令只能合并两个相邻的Region，但也能使用参数来强制合并不相邻Region，但该参数风险较大，不推荐生产线上使用
### 8.3 Region分裂
- HBase最核心的功能之一，是实现分布式可扩展性的基础
1. Region分裂触发策略（总共6种，常见的3种如下）
    - ConstantSizeRegionSplitPolicy
        - 0.94版本之前默认分裂策略
        - 一个Region中最大Store大小超过设置阈值后会触发分裂
        - 实现简单但弊端在于对于大表和小表没有明显的区分：阈值设置得大对大表比较友好，但小表可能就不会触发分裂导致只有一个Region；设置得小对小表友好，但大表会分裂出一堆Region，对集群管理、资源使用都有害
    - IncreasingToUpperBoundRegionSplitPolicy
        - 0.94-2.0版本默认分裂策略
        - 总体与前一任策略思路相同，使用阈值触发分裂，但阈值会在一定条件下不断调整
        - 调整后的阈值大小与Region所属表在当前RegionServer上的Region个数有关
        - 阈值等于#regions * #regions * #regions * flush size *２
        - 阈值不会无限增大，用户可以设置最大值
        - 能自适应大表和小表，但在大集群场景下，很多小表会产生大量小Region，分散在整个集群中
    - SteppingSplitPolicy
        - 2.0版本默认分裂策略
        - 相比前一任简单了一些
        - 阈值大小和待分裂Region所属表在当前RegionServer上的Region个数有关
        - 如果Region个数为1，阈值则为flush size * 2，否则为最大值
        - 小表不会再产生大量的小Region了
    - 一般情况下用默认分裂策略就行
2. Region分裂准备工作——寻找分裂点
- HBase对于分裂点的定义：整个Region中最大Store中的最大文件中最中心的一个Block的首个rowkey。如果定位到的rowkey是整个文件的首个rowkey或最后一个rowkey，则认为没有分裂点
- 没有分裂点的场景：待分裂Region只有一个Block时
3. Region核心分裂流程
- HBase将Region分裂包装成一个事务，以保证分裂的原子性，整个事务分为三个阶段：
    1. prepare阶段
        - 在内存中初始化两个子Region，具体为两个HRegionInfo对象
        - 同时生成一个transaction journal，用于记录分裂的进展
    2. execute阶段
        1. RegionServer将ZooKeeper节点/region-in-transition中该Region的状态更新为SPLITTING
        2. Master通过watch同一个ZooKeeper节点检测到Region状态改变，并修改内存中Region状态（在Master页面RIT模块中可见）
        3. 在父存储目录下新建临时文件夹.split，保存split后的daughter region信息
        4. 关闭父Region。父Region关闭数据写入并触发flush操作，将写入Region的数据全部持久化磁盘。此时短期内客户端落在父Region上的请求都会抛出异常NotServingRegionException
        5. 在.split文件夹下新建两个子文件夹，daughter A和B，并在文件夹中生成reference文件，分别指向父Region中的对应文件（**最核心的一步**）
            - reference文件的文件名可以指出父Region中的HFile文件
            - reference文件是一个引用文件（并非Linux链接文件），文件内容并非用户数据，而是由两部分构成：分裂点splitkey和一个boolean类型变量（用于表示该reference文件引用的是父文件的上半部分true还是下半部分false）
        6. 父Region分裂为两个子Region后，将daughter A和B拷贝到HBase根目录下，形成两个新Region
        7. 父Region通知修改hbase:meta后下线，不再提供服务。下线后父Region的meta信息不会马上删除，而是将split列、offline列标记为true，并记录两个子Region
        8. 开启daughter A和B两个子Region，通知修改meta表，正式对外提供服务
    3. rollback阶段
        - 整个分裂过程分为很多个子阶段，JournalEntryType会记录各个子阶段，并给回滚程序提供清理对应垃圾数据的依据
4. Region分裂原子性保证
    - HBase使用状态机的方式保存分裂过程中的每个子步骤状态，但它们都只存储在内存中，如果出现RegionServer宕机的情况，可能出现RIT状态，需要使用HBCK工具具体查看并分析解决方案
    - 2.0版本后会实现新的分布式事务框架Procedure V2，使用类似HLog的日志文件存储这种单机事务的中间状态（已经实现了吗？）
5. Region分裂对其它模块的影响
    - 分裂过程并不涉及数据移动，子Region的文件实际没有用户数据，仅存储一些元数据信息，如分裂点rowkey等
    - 通过reference文件查找数据的流程
        1. 根据reference文件名（父Region名+HFile文件名）定位到真实数据所在文件路径
        2. 根据reference文件内容中记录的两个重要字段确定实际扫描范围
    - 父Region的数据迁移到子Region目录的时间
        - 迁移发生在子Region执行Major Compaction时，其本质便是一次数据迁移
        - 子Region在做Major Compaction时会将父目录中属于该Region的数据读出来并写入自己的目录数据文件中
    - 父Region被删除的时间
        - Master会启动一个线程定期遍历检查所有处于splitting状态的父Region，确认其是否可以被清理
        - 检查过程分为两步：先去meta表中读出所有split列为true的Region，并加载出分裂后的两个子Region；检查两个子Region中是否还存在引用文件，如果不存在就删掉父Region
### 8.4 HBase的负载均衡应用
- 实际生产环境中，负载均衡机制最重要的应用场景是系统扩容，其一般分为两个步骤：增加节点并让系统感知到节点加入；将系统中已有节点负载迁移到新加入节点上
- 负载均衡策略需要明确系统负载是什么，通过哪些元素刻画，以及明确当前负载，并在需要的时候制订负载迁移计划
- 目前官方支持两种负载均衡策略：
    1. SimpleLoadBalancer策略
        - 能保证每个RegionServer的Region个数基本相等（所有RegionServer上的Region个数在[floor(average), ceil(average)之间]，average=#Region/#RegionServer
        - 此策略中负载就是Region个数，集群负载迁移计划就是Region从个数较多的RegionServer上迁移到个数较少的RegionServer上
        - 此策略简单易懂，但没考虑RegionServer上的读写QPS、数据量大小等因素，实际可能会忽略一些热点数据
    2. StochasticLoadBalancer策略
        - 对于负载的定义更复杂，是由多种独立负载加权计算的复合值，包括：Region个数、Region负载、读请求数、写请求数、Storefile大小、MemStore大小、数据本地率、移动代价
        - 上述各独立负载会经过加权计算得到一个代价值，系统使用这个代价值来评估当前Region分布是否均衡，越均衡代价值越低。HBase通过不断随机挑选迭代来找到一组Region迁移计划，使得代价值最小

## 第9章 宕机恢复原理
### 9.1 HBase常见故障分析
- HBase系统中主要有两类服务进程：Master进程和RegionServer进程，前者主要负责集群管理调度，一般没什么压力，后者主要负责用户的读写服务，进程中包含很多缓存组件以及与HDFS交互的组件，往往有很大压力，容易出故障
- 一些常见的可能导致RegionServer宕机的异常：Full GC、HDFS异常、物理机器宕机、HBase BUG
### 9.2 HBase故障恢复基本原理
1. Master故障恢复原理
    - HBase采用热备方式来实现Master高可用，通常要求集群中至少启动两个Master进程，进程启动后会到ZooKeeper上的Master节点进行注册，注册成功后会成为Active Master，未成功的其它进程会在Backup-Masters节点进行注册，并持续关注Active Master的情况，一旦Active Master宕机，它们会立刻得到通知并再次竞争注册Master节点
    - Active Master会接管整个系统的元数据管理任务，以及响应用户的各种管理命令
2. RegionServer故障恢复原理
    - RegionServer发生宕机时HBase会马上检测到，并将宕机RegionServer上的所有Region重新分配到集群中其他正常的RegionServer上，再通过HLog进行丢失数据恢复，完成之后就可以对外提供服务，无需人工干预，流程如下：
    1. Master检测宕机是通过ZooKeeper实现的，RegionServer会周期性向ZooKeeper发送心跳，一旦心跳停止发送并超时，则ZooKeeper认为该RegionServer宕机离线
    2. RegionServer宕机后MemStore中还未持久到文件的这部分数据必然会丢。HLog中所有Region的数据都混合存储在同一个文件中，为了使数据能够按照Region进行组织回放，需要将HLog日志进行切分再合并，将同一个Region的数据最终合并在一起
    3. Master重新分配宕机RegionServer上的Region，但还未上线
    4. 回放HLog日志补救数据
    5. 恢复完成，继续对外提供服务
### 9.3 HBase故障恢复流程
- 对于HLog中不同Region的数据切分，早期HBase的策略是整个切分过程都由Master来完成，有很大效率问题。后来使用了分布式日志切分（DLS）实现，借助了Master和所有RegionServer的计算能力进行日志切分：Master作为协调者，RegionServer作为工作者，将HLog切分并写入hdfs中
- 分布式日志回放（DLR）则在分布式日志切分的基础上做了两点改动：先重新分配Region再切分回放HLog，且Region重新分配打开后状态设置为Recovering，此状态下的Region可以对外提供写服务，但不能提供读服务，且不能split和merge等
- DLR与DLS的区别在于DLR在将HLog分解为Region-Buffer后并没有写入小文件，而是直接执行回放以减小小文件的读写IO消耗，解决DLS的短板
### 9.4 HBase故障时间优化
- HBase故障恢复的4个核心流程：故障检测、切分HLog、Assign Region、回放Region的HLog日志，其中切分HLog的耗时最长
- 早期设计中针对每个Region都会打开一个writer做数据的追加写入，但这种设计会导致HDFS集群上DataNode的Xceiver线程消耗过大（总数有限，消耗数为writer数*副本数），达到上限后会不断报错导致HBase停止服务
- 小米HBase团队提出的解决方法是通过writer池控制writer数量（HBase1.4.1及以上版本）

## 第10章 复制
- HBase默认使用异步复制，RegionServer的后台线程会不断推送HLog的Entry到Peer集群，但HBase1.X无法保证推送顺序一致，可能造成主从集群数据不一致，且备份机房数据肯定会滞后，无法满足热备需求，为了解决这两个问题HBase2.X实现了串行和同步复制
## 10.1 复制场景及原理
- Peer是指一条从主集群到备份集群的复制链路
- HBase2.x的复制管理流程：
    1. Client将创建Peer的请求发送到Master
    2. Master内实现了一个名为Procedure的框架，它会将管理操作拆分成N个步骤，每执行完一个步骤都会把状态信息持久化到HDFS，以供异常恢复后继续执行。对于Peer创建来说，Procedure会创建相关的ZNode，并将复制相关的元数据保存在ZooKeeper中
    3. Master的Procedure会向每一个RegionServer发送创建Peer的请求，直到所有RegionServer都成功创建Peer；否则会重试
    4. Master返回给HBase客户端
- HBase2.x的复制流程：
    1. 创建Peer时，每一个RegionServer会创建一个ReplicationSource线程，把当前正在写入的HLog保存在复制队列中，然后在RegionServer上注册一个Listener，用于监听HLog Roll操作，一旦操作发生，ReplicationSource会把这个HLog分到对应的walGroup-Queue中，同时把HLog文件名持久化到ZooKeeper上
    2. 每个walGroup-Queue后端辰一个ReplicationSourceWALReader线程，会不断地从Queue中取出一个HLog，然后把其中的Entry逐个读出来，放到一个名为entryBatchQueue的队列中
    3. entryBatchQueue队列后端有一个名为ReplicationSourceShipper的线程，不断地从Queue中取出Log Entry，交给Peer的ReplicationEndpoint。后者将这些Entry打包成一个replicateWALEntry操作，通过RPC发送到Peer集群的某个RegionServer上。对应Peer集群的RegionServer将replicateWALEntry解析成若干个Batch操作，并调用batch接口执行。RPC调用成功后，ReplicationSourceShipper会更新最近一次成功复制的HLog Position到ZooKeeper
## 10.2 串行复制
- 非串行复制导致的问题：当Region从一个RegionServer移到另外一个RegionServer的过程中，Region的数据会分散在两个RegionServer的HLog上，而两个RegionServer完全独立地推送各自的HLog，从而导致同一个Region的数据并行写入Peer集群
- 一个简单的解决思路：把Region的数据按照Region移动发生的时间点t0分成两段，小于t0的数据在RegionServer0的HLog上，大于t0的数据在RegionServer1的HLog上。推送时先让RegionServer0推，等它推完再让RegionServer1推，以保证顺序一致性
- 目前社区版本的实现思路大概如同上述，其中有三个重要概念：
    1. Barrier：与上述思路的t0相似，每次Region重新assign到新RegionServer时，新RegionServer打开Region前能读到的最大SequenceId（对应此Region在HLog中的最近一次写入数据分配的SequenceId）。因此每Open一次Region，就会产生一个新Barrier，N个Barrier将Region的SequenceId数轴划分为N+1个区间
    2. LastPushedSequenceId：该Region最近一次成功推送到Peer集群的HLog的SequenceId，每次成功推送一个Entry到Peer集群后，都需要将此值更新
    3. PendingSequenceId：该Region当前读到的HLog的SequenceId
- HBase只需对每个Region维护一个Barrier列表和LastPushedSequenceId即可按照规则确保数据区域推送的顺序一致
- 位置靠后的RegionServer会检查LastPushedSequenceId的值来判断当前的进度，如果没轮到自己就休眠一会之后再来检查
## 10.3 同步复制
- 设计思路：RegionServer在收到写入请求后，除了在主集群上写HLog日志，还会在备份集群上写一份RemoteWAL日志，只有等HLog+RemoteWAL+MemStore写入成功后才会返回成功给客户端。除此之外，主备集群间还会开启异步复制链路（双保险？），当主集群的HLog通过异步复制推送到备份集群后其对应的RemoteWAL才会被清理。因此RemoteWAL可以认为是成功写入主集群但未被异步复制成功推送到备份集群的数据
1. 集群复制的几种状态
    - Active：该状态下的集群将在远程集群上写RemoteWAL日志，同时拒绝接收来自其他集群的复制数据。一般情况下，同步复制中的主集群会牌Active状态
    - Downgrade Active(DA)：该状态下的集群将跳过写RemoteWAL流程，同时拒绝接收来自其他集群的复制数据。一般情况下，同步复制中的主集群因备份集群不可用卡住后会被降级为DA状态，以满足业务的实时读写
    - Standby(S)：该状态下的集群不容许Peer内的表被客户端读写，只接收来自其他集群的复制数据，并确保不会将本集群中Peer内的表数据复制到其他集群上。一般情况下，同步复制中的备份集群会处于Standby状态
    - None(N)：表示没有开启同步复制
2. 建立同步复制
    - 建立同步复制可分为三步：
    1. 在主备集群分别建立一个指向对方的同步复制Peer，此时两个集群的状态默认为DA
    2. 通过transit_peer_sync_replication_state命令将备份集群状态从DA切换成S
    3. 将主集群状态从DA切换成A
3. 备集群故障处理流程
    1. 先将主集群从Active降级到DA，以免因为写RemoteWAL失败而导致写入请求失败，后续业务的写入会通过异步复制来完成备份
    2. 确保备份集群恢复后，将备份集群状态切换为S，失败期间的数据由异步复制完成同步
    3. 把主集群状态从DA切换回A
4. 主集群故障处理流程
    1. 将备份集群状态从S切换成DA，不再接收来自主集群的数据，此时以备份集群数据为准。这个过程中备份集群会先回放RemoteWAL日志以保证主备数据一致，之后再让业务方把读写流量切换过来
    2. 主集群恢复后，还会以为自己的状态是A，但此时读写流量都在备集群上
    3. 在建立同步复制过程中，备份集群建立了一个向主集群复制的Peer，由于A状态下会拒绝来自其它集群的复制请求，因此这个Peer会阻塞客户端写向备集群的HLog。此时把主集群切换为S即可，等备集群Peer将数据同步回主集群后，数据会最终一致
    4. 把备集群状态从DA切换为A，完成主备切换
- 同步复制和异步复制对比
    - 同步复制需要占用2倍的带宽和存储空间，用于RemoteWAL的写入
    - 同步复制总能保证数据的最终一致，而异步不能
    - 若主集群故障，异步复制下服务不可用，同步复制下只需花少许时间重放RemoteWAL便可恢复服务
    - 同步复制的运维操作更复杂，需要理解集群状态并手动切换主备集群
    - 同步复制下的写入性能会稍低13%左右

## 第11章 备份与恢复
### 11.1 Snapshot概述
1. HBase备份与恢复工具的发展过程
    - 使用distcp进行关机全备份：使用Hadoop提供的文件复制工具distcp将HBase目录复制到另一个目录中。但需要关闭当前集群，不提供所有读写操作服务
    - 使用copyTable工具在线跨集群备份：通过MapReduce程序全表扫描数据并写入另一个集群。不需要关闭源集群，但会极大增加服务压力，并且耗时长，只能保证行级一致性
    - 使用Snapshot在线备份：以快照技术为基础原理，不需要拷贝任何数据，速度快，几乎对服务无影响，能保证数据一致性
2. 在线Snapshot备份能实现什么功能
    - 全量/增量备份：可以在异常发生时快速回滚到指定快照点
    - 数据迁移：可以使用ExportSnapshot功能将快照导出到另一个集群，实现数据迁移（如将数据导出到HDFS供Hive/Spark等离线OLAP进行分析）
3. 在线Snapshot备份与恢复的用法
    - snapshot：为表打一个快照，但不涉及数据移动
    - restore_snapshot：用于恢复指定快照，恢复过程会替代原有数据，快照点之后的所有更新会丢失
    - clone_snapshot：根据快照恢复出一个新表，不涉及数据移动
    - ExportSnapshot：将集群A的快照数据迁移到集群B，是HDFS层面的操作，使用MapReduce进行数据的并行迁移，Master和RegionServer并不参与，不会带来额外内存与GC开销。但DataNode在拷贝数据时需要额外带宽及IO负载（可以用参数限制带宽）
### 11.2 Snapshot创建
#### 11.2.1 Snapshot技术原理
- Snapshot机制并不拷贝数据，因为在HBase的LSM树类型系统结构下数据只会被不断地追加，因此实现某个表的Snapshot只需为当前表的所有文件分别新建一个引用。对于其它新写入的数据，重新创建一个新文件写入即可
- Snapshot流程中需要将MemStore中的缓存数据flush到文件中，之后为所有HFile文件分别新建引用指针，这些指针元数据就是Snapshot
#### 11.2.2 在线Snapshot的分布式架构——两阶段提交
- Snapshot需要保证分布在多个RegionServer上的Region要么全部完成Snapshot，要么都不做，不能出现中间状态
- Snapshot的两阶段提交实现：
- prepare阶段：
    1. Master在ZooKeeper创建一个/acquired-snapshotname节点，并在此节点上写入Snapshot相关信息
    2. 所有RegionServer监测到这个节点，根据其携带的Snapshot表信息查看当前RegionServer上是否存在目标表，如果存在则遍历目标表中的所有Region，针对每个Region分别Snapshot，操作结果写入临时文件夹
    3. RegionServer执行完成后在/acquired-snapshotname节点下新建一个子节点/acquired-snapshotname/nodex，表示nodex节点完成了Snapshot准备工作
- commit阶段：
    1. 一旦所有RegionServer完成Snapshot并建立了相应节点，Master便认为准备工作完成，会新建一个/reached-snapshotname节点，表示发送一个commit命令给参与的RegionServer
    2. 所有RegionServer监测到这个节点后会执行commit操作，也就是将临时文件夹中的数据移动到最终文件夹
    3. RegionServer在/reached-snapshotname节点下新建子节点/nodex表示节点nodex完成了Snapshot工作
- abort阶段：
    - 如果一定时间内/acquired-snapshotname节点个数没有满足条件，则认为准备工作超时，Master会新建另一节点/abort-snapshotname，所有RegionServer监听到后会清理临时文件夹中的Snapshot数据
- 可以看出在此过程中，Master充当了协调者，RegionServer充当了参与者，ZooKeeper则是二者之间沟通的桥梁和事务状态记录者
#### 11.2.3 Snapshot核心实现
- 每个Region实现Snapshot的流程：
    1. 将MemStore数据flush到HFile
    2. 将region info元数据记录到Snapshot文件中
    3. 将Region中所有HFile文件名记录到Snapshot文件夹中
- Master会在所有Region完成Snapshot后执行一个汇总（consolidate）操作，将所有region snapshot manifest汇总成一个单独manifest，此文件可以在HDFS目录下找到：/hbase/.hbase-snapshot/snapshotname/data.manifest
### 11.3 Snapshot恢复
- 以clone_snapshot为例，其流程如下：
    1. 预检查，确认当前表没有执行snapshot以及restore等操作，否则返回错误
    2. 在tmp文件夹下新建目标表目录并在表目录下新建.tabledesc文件，在其中写入表schema信息
    3. 新建region目录，根据snapshot manifest中的信息新建Region相关目录以及HFile文件
    4. 将表目录从tmp文件夹下移到HBase Root Location
    5. 修改hbase:meta表，将克隆表的信息添加进其中（二者的Region名会不一样，因为表名不一样了）
    6. 将这些Region通过round-robin方式均匀分配到整个集群中，并在ZooKeeper上将克隆表状态设为enabled，正式对外提供服务
- Snapshot用一种名为LinkFile的文件指向原文件，其中并不包含任何数据，但其文件名可以直接定位到原始文件的具体路径（原始文件表名+引用文件所在Region+引用文件名）
### 11.4 Snapshot进阶
- 原始表发生了Compaction时，会将原始表数据复制到archive目录下（不删除），因此Snapshot的引用链仍然有效，只是路径变了
- 当普通表被删除一段时间后，archive中的数据也会被删除，因为Master上有一个定期清理archive中垃圾文件的线程
- Snapshot原始表进入archive后不会被删除，因为archive目录下会生成反向引用文件来帮助原始表文件找到引用文件，以此知道自己还不能死
- 新表在执行compact的时候会将合并后的文件写入到新目录并将相关的LinkFile删除，此时新目录中就是真实数据而不再是引用了