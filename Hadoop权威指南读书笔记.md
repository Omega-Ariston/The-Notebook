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
    - 在Hadoop集群上运行作业时，要把代码打包为一个jar文件以供Hadoop在集群上发布，但不必明确指定jar文件名称，通过Job对象的setJarByClass()方法传递一个类即可，Hadoop会利用其寻找包含它的jar文件
    - FileInputFormat类的静态方法addInputPath()可以定义输入数据的路径，可以是单个文件或目录（此时，目录下所有文件都会被输入），或符合特定pattern的一系列文件。亦可通过多次调用此方法进行多路径输入
    - FileOutputFormat类的静态方法setOutputPath()可用于指定reduce函数输出文件的写入目录。此目录在运行前不应存在，否则Hadoop会报错，目的是为了防止数据丢失（结果被意外覆盖）
    - setMapperClass()和setReducerClass()方法用于指定map类与reduce类
    - setOutputKeyClass()和setOutputValueClass()方法控制reduce函数的输出类型，且必须与reduce类产生的相匹配。map函数的输出类型默认与reduce函数相同，如若不同，则必须通过setMapOutputKeyClass()与setMapOutputValueClass()来设置输出类型
    - 输入的格式通过InputFormat来控制，默认为TextInputFormat
    - waitForCompletion()方法提交作业并等待执行完成，其返回的布尔值用于表示执行的成功与失败
- 如果调用Hadoop命令的第一个参数是一个类名，Hadoop会启动一个JVM来运行这个类，该命令会将Hadoop库（及其依赖关系）添加至ClassPath中，同时也能获取配置信息。为了将应用类添加到类路径，Hadoop定义了一个HADOOP_CLASSPATH环境变量，由Hadoop脚本执行相关操作
- 作业（job）：客户端执行的工作单元，包括输入数据、MapReduce程序和配置信息。Hadoop将job分成若干个task，包括两类：map task、reduce task，这些task运行在集群节点上，由Yarn进行调度，失败时会在另一个节点上重新运行
- Hadoop将MapReduce输入数据划分为等长的小数据块，称为数据分片（Input Split），Hadoop为每个分片构建一个map任务
- 分片被切分得更细时负载平衡的质量会更高，但管理分片的总时间和构建map task的开销会决定作业的整个执行时间。合理的大小趋向于一个HDFS块的大小（默认128MB，可针对集群调整新建文件大小或创建文件时单独指定），因为当分片横跨两个数据块时，同一个HDFS节点基本不可能同时存储两块数据
- Hadoop会在存储有输入数据的HDFS节点上运行map任务以提高数据本地化。但有时一个map任务的输入分片所处的HDFS节点可能正在运行其它任务，此时本地化会退化为同机架，甚至不同机架（非常偶然），因此会涉及数据的网络传输
- map任务将其输出的中间结果写入本地磁盘，并由reduce任务处理后产生最终结果，一旦完成作业，这些信息就可以删除。当中间结果传递失败时，Hadoop将在另一个节点上重新运行map任务
- reduce任务并不具备数据本地化优势，因为其输入通常来自所有mapper的输出，因此会涉及数据的网络传输。对于reduce输出的HDFS块，第一个副本会存储在本地节点上
- combiner通过Reducer类来定义，在job中通过setCombinerClass()方法设置，其属于优化方案，因此Hadoop无法确定要对一个指定的map任务输出记录调用多少次combiner，但不管调用多少次，reducer的输出结果都应是一样的