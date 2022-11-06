# Hadoop权威指南读书笔记
## 第一部分 Hadoop基础知识
### 第一章 初识Hadoop
- 不使用配置大量硬盘的数据库进行大规模数据分析，而是使用Hadoop，因为当今计算机磁盘的发展趋势：寻址时间的提升远远不敌于传输速率的提升
- RDBMS：适用于索引后数据集的点查询与更新，适合持续更新的数据集
- MapReduce：适合解决需要以批处理方式分析整个数据集的问题
- Hadoop对非结构化或半结构化数据非常有效，因为它在处理数据时才对其进行解释，提供灵活性的同时避免了RDBMS数据加载阶段带来的开销（只需文件拷贝）
- 关系型数据库往往是规范（Normalized）的，以保持数据的完整与不冗余，但对Hadoop而言，会使记录读取变得非本地化
- Web服务器日志是典型的非规范化数据（同一客户端全名可能多次出现），为Hadoop非常合适分析各种日志文件的原因之一

### 第二章 关于MapReduce
- Hadoop本身提供了一套可优化网络序列化传输的基本类型，而不直接使用Java的内嵌类型。这些类型位于org.apache.hadoop.io包中，如对应Long的LongWritable，对应String的Text，和对应Integer的IntWritable
- map()方法还提供Context实例用于输出内容的写入
- Job对象用于指定作业执行规范的流程：
    1. 在Hadoop集群上运行作业时，要把代码打包为一个jar文件以供Hadoop在集群上发布，但不必明确指定jar文件名称，通过Job对象的setJarByClass()方法传递一个类即可，Hadoop会利用其寻找包含它的jar文件
    2. FileInputFormat类的静态方法addInputPath()可以定义输入数据的路径，可以是单个文件或目录（此时，目录下所有文件都会被输入），或符合特定pattern的一系列文件。亦可通过多次调用此方法进行多路径输入
    3. FileOutputFormat类的静态方法setOutputPath()可用于指定reduce函数输出文件的写入目录。此目录在运行前不应存在，否则Hadoop会报错，目的是为了防止数据丢失（结果被意外覆盖）
    4. setMapperClass()和setReducerClass()方法用于指定map类与reduce类
    5. setOutputKeyClass()和setOutputValueClass()方法控制reduce函数的输出类型，且必须与reduce类产生的相匹配。map函数的输出类型默认与reduce函数相同，如若不同，则必须通过setMapOutputKeyClass()与setMapOutputValueClass()来设置输出类型
    6. 输入的格式通过InputFormat来控制，默认为TextInputFormat
    7. waitForCompletion()方法提交作业并等待执行完成，其返回的布尔值用于表示执行的成功与失败
- 如果调用Hadoop命令的第一个参数是一个类名，Hadoop会启动一个JVM来运行这个类，该命令会将Hadoop库（及其依赖关系）添加至ClassPath中，同时也能获取配置信息。为了将应用类添加到类路径，Hadoop定义了一个HADOOP_CLASSPATH环境变量，由Hadoop脚本执行相关操作
- 作业（job）：客户端执行的工作单元，包括输入数据、MapReduce程序和配置信息。Hadoop将job分成若干个task，包括两类：map task、reduce task，这些task运行在集群节点上，由Yarn进行调度，失败时会在另一个节点上重新运行
- Hadoop将MapReduce输入数据划分为等长的小数据块，称为数据分片（Input Split），Hadoop为每个分片构建一个map任务
- 分片被切分得更细时负载平衡的质量会更高，但管理分片的总时间和构建map task的开销会决定作业的整个执行时间。合理的大小趋向于一个HDFS块的大小（默认128MB，可针对集群调整新建文件大小或创建文件时单独指定），因为当分片横跨两个数据块时，同一个HDFS节点基本不可能同时存储两块数据
- Hadoop会在存储有输入数据的HDFS节点上运行map任务以提高数据本地化。但有时一个map任务的输入分片所处的HDFS节点可能正在运行其它任务，此时本地化会退化为同机架，甚至不同机架（非常偶然），因此会涉及数据的网络传输
- map任务将其输出的中间结果写入本地磁盘，并由reduce任务处理后产生最终结果，一旦完成作业，这些信息就可以删除。当中间结果传递失败时，Hadoop将在另一个节点上重新运行map任务
- reduce任务并不具备数据本地化优势，因为其输入通常来自所有mapper的输出，因此会涉及数据的网络传输。对于reduce输出的HDFS块，第一个副本会存储在本地节点上
- combiner通过Reducer类来定义，在job中通过setCombinerClass()方法设置，其属于优化方案，因此Hadoop无法确定要对一个指定的map任务输出记录调用多少次combiner，但不管调用多少次，reducer的输出结果都应是一样的

### 第三章 Hadoop分布式文件系统
- HDFS的构建思路：一次写入、多次读取是最高效的访问模式。每次分析都将涉及该数据集的大部分数据甚至全部，因此读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要
- HDFS为高数据吞吐量应用优化，可能会以提高时间延迟为代价，因此要求低延迟数据访问的应用（例如几十毫秒范围）不适合在HDFS上运行
- HDFS的文件系统所能存储的文件总数受限于namenode的内存容量（每个文件、目录和数据块的存储信息大约占150字节）
- HDFS的文件写入只支持单个写入者，且是以“只添加”方式在文件末尾写数据
- HDFS的文件被划分为多个块（chunk），作为独立的存储单元。但与单磁盘文件系统不同，HDFS中小于块大小的文件不会占据整个块的空间
- namenode用于维护文件系统树及整棵树内所有的文件和目录。这些信息以两个文件形式永久保存在本地磁盘上：命名空间镜像文件和编辑日志文件
- namenode也记录每个文件中各个块所在的数据节点信息，但并不永久保存块的位置信息，因为系统启动时会根据数据节点信息重建
- datanode会定期向namenode发送自己存储的块的列表
- namenode的两种容错机制：
    1. 备份文件系统元数据持久状态文件，使namenode在多个文件系统上保存元数据持久状态，对它们的写操作实时同步且具有原子性。一般是通过远程挂载的网络文件系统（NFS）来完成
    2. 在另一台机器上运行一个辅助namenode，定期合并编辑日志与命名空间镜像，以防止编辑日志过大。它会保存合并后的命名空间镜像副本，并在namenode节点发生故障时启用，但由于信息的滞后性，主备转换时可能会丢失部分数据。在这种情况下，一般把存储在NFS的namenode元数据复制到辅助namenode并作为新的namenode运行
- 对于访问频繁的文件，其对应的块可能会被datanode显式缓存在内存中，以堆外块缓存的形式存在。用户或应用可以通过在缓存池（用于管理缓存权限和资源使用的管理性分组）中增加一个cache directive来告诉namenode需要缓存哪些文件及存多久
- 联邦HDFS：允许namenode通过分治文件系统命名空间的方式进行横向扩展，每个namenode维护一个命名空间卷（namespace volumn），由命名空间的元数据和一个数据块池组成，数据块池包含该命名空间下文件的所有数据块。卷之间相互独立不通信，单独失效也不影响其它。数据池块不再进行切分，集群中的datanode需要注册到每个namenode，并存储来自多个数据块池中的数据块
- 不启用HA时从失效的namenode中恢复：
    1. 启动一个拥有文件系统元数据副本的新namenode
    2. 将命名空间映像导入内存
    3. 重演编辑日志
    4. 接收到足够多的来自datanode数据块报告并退出安全模式
- Hadoop2引入的HA支持：
    - 配置了一对active-standby的namenode
    - namenode之间需要通过高可用共享存储实现编辑日志的共享，备用namenode接管后，会通读共享编辑日志直至末尾，并继续读取由活动namenode写入的新条目
    - datanode需要同时向两个namenode发送数据块处理报告，因为数据块映射信息存储于内存，而非磁盘
    - 客户端需要使用特定机制处理namenode的失效问题，且对用户透明
    - 辅助namenode的角色被备用namenode包含，其亦为活动namenode的命名空间设置周期性检查点
    - 可供选择的两种高可用共享存储：NFS或群体日志管理器QJM
    - QJM以一组日志节点的方式运行，每一次编辑必须写入多数日志节点（与ZooKeeper的工作方式类似）
    - 活动namenode失效后，备用namenode能快速（几十秒）实现任务接管，因为最新状态存储在内存中，包括最新的编辑日志条目和最新的数据块映射信息。实际观察到的时间会稍长（1分钟左右），因为系统需要保守确定活动namenode是否真的失效了
    - 活动namenode+备用namenode都失效的情况下（很罕见），依旧可以声明一个备用namenode并实现冷启动
    - 故障转移控制器：每个namenode都运行了一个轻量级的控制器，通过心跳机制监视宿主namenode是否失效，并在失效时进行故障切换（亦可被手动启用）
    - 在非平稳故障转移的情况下（如网络慢或被分割），同样也可能激发故障转移，但是先前的活动namenode仍然保持活动，这会导致其响应并处理客户过时的读请求（因为QJM同一时间仅允许一个namenode向编辑日志中写入数据）。因此需要设置一个SSH规避命令用于杀死namenode的进程
    - 对用户而言，可以通过配置客户端配置文件实现故障转移的控制：HDFS URI使用一个逻辑主机名，映射到一对namenode地址，客户端类库会访问每一个namenode直到处理完成
- Java接口部分暂时跳过
- 客户端通过远程调用访问namenode以获得文件块地址，namenode会返回存有该文件块副本的datanode地址，并且这些datanode会根据它们与客户端的距离进行排序（根据网络拓扑）
- 客户端调用read()方法时DFSInputStream会进行datanode的连接，并在到达块末端时关闭连接，并寻找下一个块最佳的datanode（用户无感知）
- 读取数据时若碰到datanode故障，DFSInputStream会从这个块的下一个最邻近datanode中读取数据，并记住故障datanode以保证以后不从它那读。如果发现损坏的块，则也会从其它datanode上读副本并将错误块通知给namenode
- 客户端写入数据时，DFSOutputStream会将它分为一个个数据包，并写入内部队列，DataStreamer负责处理数据队列，挑选出合适的一组datanode用于存储数据，并要求namenode进行数据块的分配。DataStreamer会将数据先发往第一个副本的datanode，该datanode存储数据包后会将其发送给第2个datanode，依此类推，写入成功后datanode会向DFSOutputStream发送ack，全部ack后才删队列
- 当写入数据过程中datanode故障时，关闭数据写入管线，将队列中所有数据包重新添加回队列最前端，以确保故障节点的下游datanode不漏数据。为存储在另一正常datanode的当前数据块指定一个新标识并传送给namenode，以便故障恢复后出错节点可以删除存储的部分数据块（？）。之后从管线中删除故障节点，并以剩余节点重建管线，完成写入。namenode注意到副本数不足时会在另一个节点上创建一个新的副本，后续数据块正常接受处理
- 块数据可以在集中异步复制，直到达到目标副本数才算写入成功
- 调用close()时会将剩余数据包都写入datanode管线，并在联系namenode告知其文件写入完成前，等待确认（此时namenode已经知道文件由哪些块组成，只需等待数据块复制即可）
- **HDFS的一致模型**：当前正在写入的块对其它reader不可见，除非调用hflush()方法将所有缓存强制刷新到datanode中以使数据对新reader可见（hflush仅能保证数据到达节点内存中，不保证已落盘，因此无法避免断电带来的数据丢失，用hsync就可以保证落盘）。HDFS中的关闭文件隐含了hflush()方法
- distcp会使用MapReduce任务来完成文件的复制（仅需mapper），每个文件用一个mapper复制，并且distcp会尽量把每个map的数据大小分配得大致相等（把文件分块），可以通过-m把map数改为多于集群中节点数，以使数据分布均衡（第1个副本会写在map本地节点）。或使用balancer来改善集群中块的分布