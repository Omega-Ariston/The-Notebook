# 从0到1学会Apache Flink读书笔记
## 第一章 Runtime核心机制剖析
- Flink Runtime：Flink提供的统一的分布式作业执行引擎，这之上提供了Data Stream和DataSet两套API，分别用于编写流作业与批作业，以及一组更高级的API来简化特定作业的编写
- Runtime采用Master-Slave结构
- Master包含三个组件（AM进程中）：
    - Dispatcher：负责接收用户提供的作业，为其拉起一个新的JobManager组件
    - ResourceManager：负责资源的管理，整个Flink集群只有一个RM
    - JobManager：负责管理作业的执行，每个作业都有自己的JM
- 基于上述结构的作业提交过程：
    - 提交脚本启动一个Client进程负责作业的编译与提交
    - Client将用户代码编译为一个JobGraph，过程中会检查与优化
    - Client将JobGraph提交到集群中执行
    - 若AM已启动（Session模式），Client直接与Dispatcher建立连接提交作业
    - 若AM未启动（Per-Job模式），Client先向资源管理系统（Yarn、K8S）申请资源启动AM，再向其中Dispatcher提交作业
    - Dispatcher收到作业后，会启动一个JobManager组件
    - JobManager向ResourceManager申请资源启动任务
    - 若RM已有记录TaskExecutor注册的资源（Session模式），可以直接选取空闲资源进行分配。若无（Per-Job），RM也需要首先向外部资源管理系统申请资源来启动TaskExecutor，然后等待TaskExecutor注册相应资源后再继续选择空闲资源进程分配（TaskExecutor的资源通过Slot描述）
    - RM选择到空闲的Slot后会通知相应的TM将该Slot分配给哪个JM
    - TaskExecutor进行相应记录后，向JM注册
    - JM收到TaskExecutor注册的Slot后，进行实际的Task提交
    - TaskExecutor收到JM提交的Task后，会启动一个线程用于执行Task
- Per-Job模式下AM和TaskExecutor按需申请，适合长时作业，对稳定性要求高，且对资源申请时间不敏感
- Session模式适合规模小、执行时间短的作业
- 资源管理
    - ResourceManager中有个子组件SlotManager，维护当前集群中所有TaskExecutor上的Slot信息与状态（位于哪个TaskExecutor中、是否空闲等）
    - TaskExecutor启动后，会通过服务发现找到当前活跃的RM并进行注册，注册信息包含所有Slot的信息
    - RM收到注册后会让SlotManager记录Slot信息，以便日后JM来申请资源时可以从中选取Slot进行分配（按一定规则）
    - 分配完成后，RM向TM发送RPC请求要求其将指定Slot分给指定JM
    - TM如果没执行过该JM的任务，则需要先向该JM建立连接，然后发送提供slot的RPC请求
    - JM中，所有Task的请求会缓存到SlotPool中，当有Slot被提供时，从缓存池里取出相应的请求并结束其请求过程
    - Task结束后会通知JM结束状态（是否有异常），然后TM将slot标记为已占用但未执行任务。JM也会把Slot放进SlotPool里，但不会立即释放，以免需要重启任务时要重新申请Slot
    - SlotPool里缓存的Slot在超过指定时间后JM才会申请释放，与申请Slot一样，先通知TM来释放slot，然后TaskExecutor通知RM该slot已释放
    - TM与RM，TM与JM之间会心跳保活
    - Share Slot机制可以允许每个Slot中部署来自不同JobVertex的Task，以提高资源利用率，并做一点简单的负载均衡
    - JM会对JobGraph按并发展开得到ExecutionGraph，其会对每个任务的中间结果等均创建对应的对象，从而可以维护这些实体的信息与状态
- 作业调度
    - Eager：作业启动时申请资源将所有Task调度起来，一般用于没有终止的流作业
    - Lazy From Source：从Source开始按拓扑顺序按需调度，前面任务结果产出后再起下游任务（中间结果需要内存缓存或落磁盘）
- 错误恢复
    - Task执行错误
        - Restart-all：直接重启所有Task，可以通过Checkpoint从最近一次检查点继续执行，适合于流作业
        - Restart-individual：当Task间没有数据传输的时候可以直接重启出错的任务
    - 对于批作业而言，可以使用Regional重启的策略。一条Pipeline作业处理线就是一个Region（两端以批作业分隔），因为region之间数据会缓存，因此regin可以单独重启。此时ExecutionGraph可以划分为多个子图（有点像MapReduce里不同的Stage）
    - 在Region策略中，如果上游输出结果时出现问题（如TaskExecutor异常退出等），则还需要重启上游Region来重新产生相应数据。若上游输出数据的分发方式不确定（KeyBy、Broadcast属于确定，Rebalance、Random属于不确定），则为了保证结果正确性，还需要把上游region对应的所有下游region也一块重启
    - Flink集群支持多个Master备份，如果Master挂了，可以通过Zookeeper重新选主并接管工作，但为了准确维护作业状态，需要重启整个作业（有改进空间）

## 第二章 时间属性深度解析
- Flink API大体分为三个层次：底层ProcessFunction、中层Data Stream API、上层SQL/Table API。每层都依赖时间属性，但中层因封闭原因接触到时间的地方不多
- 判断使用Event Time还是Processing Time：当应用遇到问题需要从Checkpoint恢复时，是否希望结果完全相同？是的话选择Event Time，可以接受不同就选择Processing Time，后者处理起来更简单
- Processing Time使用本地节点时间，因此肯定是递增的，意味着消息有序，数据流有序
- 如果单条数据之间乱序，就考虑对整个序列进行更大程度的离散化。也就是用更“大”的眼光去看待一批一批的数据，进行时间上的大粒度划分，保证划分后的时间区块之间有序。这时在区块间加入标记（特殊处理数据），叫做watermark，表示以后到来的数据不会再小于这个时间了
- Timestamp分配与Watermark生成
    - Flink支持两种watermark生成方式
    1. 从SourceFunction产生，从源头就带上标记。通过调用collectWithTimestamp方法发送带时间戳的数据，或调用emitWatermark产生一条watermark，表示接下来不会有时间戳小于这个数值的记录
    2. 使用DataStream API时指定，调用assignTimestampAndWatermarks方法，传入不同的timestamp和watermark生成器（watermark由时间驱动或事件驱动，每次分配timestamp时都会调用watermark生成方法）
    - 建议生成的工作尽量靠近数据源
- Watermark传播
    - Watermark以广播的形式在算子之间进行传播
    - Long.MAX_VALUE表示不会再有数据
    - 单输入取其大，多输入取其小（短板效应）
    - 局限在于，即便在单一输入流的情况下，如果该输入流来自多个流Join的结果，则有可能来自不同流的消息存在时间上的巨大差异，导致快流需要等慢流，就要缓存快流的数据，是一个很大的性能开销
- ProcessFunction
    - Watermark在任务里的处理逻辑分为内部逻辑与外部逻辑，后者由ProcessFunction体现
    - 可以根据当前系统使用的时间语义去获取正在处理的记录的事件时间或处理时间
    - 可以获取当前算子的时间（可以理解为watermark）
    - 可以注册timer，允许watermark达到某个时间点时触发定时器的回调逻辑。比如实现缓存的自动清除
- Watermark处理
    - 当算子实例收到Watermark时
    1. 首先要更新当前的算子时间，以便让ProcessFunction获取
    2. 遍历计时器队列，查看是否有事件需要触发
    3. 遍历计时器的队列，逐一触发用户定义的回调逻辑
- Table API中的时间
    - 需要把时间属性提前放到表的schema中（物化）
    - 时间列和Table操作：
        1. Over窗口聚合
        2. GroupBy窗口聚合
        3. 时间窗口连接
        4. 排序
    - 这些操作必须在时间列上进行（数据流的一过性）或优先在时间列上进行（如按照其它列进行排序时必须先按时间排序），保证内部产生的状态不会无限增长下去（最终前提）

## 第三章 Checkpoint原理剖析与应用实践
- Checkpoint是从source触发到下游所有节点完成的一次全局操作
- State则是Checkpoint所做的主要持久化备份的主要数据
- State的两种类型
    1. keyed state：只能应用于KeyedStream的函数与操作中，如Keyed UDF，window state，每一个key只能属于某一个keyed state
    2. operator state：又称为non-keyed state，每个state仅与一个operator实例绑定，常见的有source state，比如记录当前source的offset
- State的另一种分类方式
    1. Managed State（推荐）：由Flink管理的State，包含上面分类的两种
    2. Raw State：Flink仅提供stream可以进行存储数据，对flink而言这些state只是一些bytes
- State的使用
    - keyed state：通过RuntimeContext.getState()方法访问，支持update()操作
    - operator state：通过FunctionInitializationContext.getOperatorStateStore().getListState()方法访问，并在snapshotState()方法中将状态写入
- State的存储
    - statebackend的分类：
        1. MemoryStateBackend
        2. FsStateBackend
        3. RocksDBStateBackend
    - 1和2运行时存储于Java堆内存中，执行Checkpoint时2才会将数据以文件格式持久化到远程存储。3则借用了RocksDB对state进行保存（内存磁盘混合）
    - 对于内存中state存储的两种实现方法：
        1. 支持异步Checkpoint（默认）：存储格式CopyOnWriteStateMap
        2. 仅支持同步Checkpoint：存储格式NestedStateMap
    - HeapKeyedStateBackend中Checkpoint序列化数据阶段默认有最大5MB数据的限制
- Checkpoint的执行机制
    1. JM中的Checkpoint Coordinator负责发起Checkpoint，向所有Source节点trigger Checkpoint
    2. source节点向下游广播barrier（实现Chandy-Lamport快照算法的核心），下游的task只有收到所有input的barrier才会执行相应checkpoint
    3. 当task完成state备份后，将备份数据的地址（state handle）通知给Checkpoint Coordinator
    4. 下游的sink节点收到所有input的barrier后会执行本地快照，将数据全量刷到持久化备份中并将State handle返回给Checkpoint Coordinator
    5. 当Checkpoint Coordinator收到所有task的state handle，就认为这次checkpoint已全局完成，并向持久化存储中备份一个checkpoint meta文件
- Checkpoint的Exactly-once语义
    - 需要在对不同输入源传来的barrier进行对齐的阶段将收到的数据缓存起来，等对齐完再处理
    - 对于At-Least-Once来说，无需缓存收集到的数据，而是直接做处理，所以导致restore时可能有数据被重复处理
    - Flink只能保证计算过程Exactly-Once，端到端则需要Source和Sink支持
- Checkpoint与Savepoint区别：
    - 概念：Checkpoint是自动容错机制；Savepoint是程序全局状态镜像
    - 目的：Checkpoint是程序自动容错、快速恢复；Savepoint是程序修改后继续从状态恢复，程序升级等
    - 用户交互：Checkpoint下Flink系统行为，Savepoint是用户触发
    - 状态文件保留策略：Checkpoint默认程序删除（可配置），Savepoint会一直保存到用户删除
