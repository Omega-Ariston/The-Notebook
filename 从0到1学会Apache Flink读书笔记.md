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
    - 用户交互：Checkpoint是Flink系统行为，Savepoint是用户触发
    - 状态文件保留策略：Checkpoint默认程序删除（可配置），Savepoint会一直保存到用户删除

## 第四章 Flink on Yarn/K8S原理剖析及实践
- Flink架构概览——Job
    - 用户通过DataStream API、DataSet API、SQL和Table API编写Flink任务，会生成一个JobGraph
    - JobGraph由source、map()、keyBy()/window()/apply()和Sink等算子组成
    - JobGraph提交给Flink集群后，能以Local、Standalone、Yarn和Kubernetes四种模式运行
- Flink架构概览——JobManager
    - 负责将JobGraph转换为Execution Graph并最终拿来运行
    - Scheduler组件负责Task的调度
    - Checkpoint Coordinator组件负责协调整个任务的Checkpoint，包括开始与完成
    - 通过Actor System与TaskManager通信
    - 其它功能，例如Recovery Metadata，用于进行故障恢复时，可以从Metadata里读取数据
- Flink架构概览——TaskManager
    - 负责具体任务的执行过程，在JM申请到资源后开始启动
    - 主要组件：
        - Memory&I/OManager，内存与IO管理
        - Network Manager，对网络方面进行管理
        - Actor system，负责网络通信
    - 被分为很多个TaskSlot，每个任务都要运行在slot里面，是调度资源的最小单位
- Flink的Standalone模式
    - Master和TM可以运行在同一台机器上，也可以不同
    - Master进程中，Standalone ResourceManager负责对资源进行管理。用户提交JobGraph给Master时要先经过Dispatcher
    - Dispatcher收到请求后会生成一个JM，然后JM向RM申请资源，再启动TM
    - TM启动后会向JM注册，之后JM再将具体的Task发给TM执行
- 运行时相关组件：Client->JobManager->TaskManager
- Flink on Yarn原理
    - Yarn任务执行过程：
        1. Client端提交任务给RM
        2. RM通知NM启动Container，并在其上启动AM
        3. AM启动后向RM注册，并重新申请资源
        4. RM向AM分配资源，AM将具体的Task调度执行
    - Yarn集群中的组件：
        - RM：负责处理客户端请求、启动/监控AM、监控NM、资源分配与调度、包含Scheduler与ASM
        - AM：运行于Slave机上，负责数据切分、申请资源和分配、任务监控和容错
        - NM：运行于Slave机上，用于单节点资源管理、AM/RM通信及汇报状态
        - Container：对资源进行抽象，包括内存、CPU、磁盘、网络等
    - Per-Job模式工作流程：
        1. Client提交Yarn App，比如JobGraph或JARs
        2. RM申请一个Container，用于启动AM，AM进程中运行Flink程序，即Flink-Yarn RM以及JobManager
        3. F-YRM向RM申请资源，申请到之后启动TM
        4. TM启动后向FYRM注册自己，成功后JM将具体任务发给TM
        - （这个模式下似乎没有Dispatcher）
    - Session模式工作流程：
        - 这个模式下Client会直接提交一个Flink-Session作为Yarn App
        - AM进程里面会包含可复用的F-YRM和Dispatcher（用于直接接收Client发来的Job）
        - Dispatcher收到请求后会在AM中启动对应的JM，多个请求会对应多个JM，每个JM在任务结束后资源不会释放
        - 多个JM共用一个Dispatcher和F-YRM
    - Yarn模式优点
        - 资源的统一管理和调度：以Container作为调度单位，可自定义调度策略
        - 资源隔离：Container资源抽象
        - 自动failover处理：NM监控及AM异常恢复
    - Yarn模式缺点
        - 资源分配静态，无法根据负载返还或扩展资源
        - 所有的container大小固定，无法区分CPU密集型与IO密集型作业
        - 作业管理页面会在作业完成后消失不可访问
- Flink on Kubernetes暂时跳过

## 第五章 数据类型和序列化
- Flink中数据类型分类：
    - 基础类型（Basic）：所有Java基础类型（装箱或不装箱）
    - 数组（Arrays）：基础类型数组或对象数组
    - 复合类型（Composite）
    - 辅助类型（Auxiliary）：Flink Java Tuple、Scala Tuple、Row、POJO
    - 泛型和其他类（Generic）：由Kryo提供序列化支持
- TypeInformation是Flink类型系统的核心类，Flink中每一个具体的类型都对应了一个TypeInformation的实现类
- TypeInformation亦可用于生成对应类型的序列化器TypeSerializer并用于执行语义检查，比如当字段作为join或grouping的键时，检查这些字段是否在该类型中存在
- 对于大多数类型，Flink可以自动生成对应的序列化器，除了GenericTypeInfo需要靠Kryo。复合类型的序列化器同样是复合的
- 实现TypeComparator可以定制类型的比较方式
- Flink的序列化过程比Java的要省空间（省去类型信息）
- 序列化的结果由Memory Segment支持，其为一个固定长度的内存，是Flink中最小的内存分配单元，相当于Java的Byte数组，每条记录都会以序列化的形式存在一个或多个Memory Segment中
- 关于序列化需要注意的几个场景：
    - 注册子类型：如果函数签名只描述了超类型，而实际执行过程中使用了子类型，让Flink了解子类型会大大提高性能。可以通过调用StreamExecutionEnvironment或ExecutionEnvironment的registertype()方法注册子类型信息
    - 注册自定义序列化器：对于不适用于自己序列化框架的数据类型，Flink让Kryo帮忙，但有的类型无法与Kryo无缝连接，需要注册
    - 添加类型提示：有时Flink无法推测出泛型信息，需要用户传入TypeHint类型提示（通常只在Java API中需要）
    - 手动创建TypeInformation：某些API调用中是必须的，因为Java的泛型类型擦除机制
- Flink无法序列化的类型会默认交给Kryo处理，如果Kryo还无法处理，有两个方案：
    1. 强制使用Avro来代替Kryo：`env.getConfig().enableForceAvro();`
    2. 为Kryo增加自定义的Serializer以增强其功能：`env.getConfig().addDefaultKryoSerializer(clazz, serializer);`
- 亦可通过`Kryoenv.getConfig().disableGenericTypes()`来彻底禁用Kryo，但此时无法处理的类会导致异常
- 通信层的序列化：
    - Task之间如果需要跨网络传输数据记录，就需要将数据序列化之后写入 NetworkBufferPool，下层的Task读出之后再进行反序列化操作
    - Flink 提供了数据记录序列化器（RecordSerializer）与反序列化器（RecordDeserializer）以及事件序列化器（EventSerializer）来保证上述过程正确执行
    - Function 发送的数据被封装成 SerializationDelegate，它将任意元素公开为IOReadableWritable以进行序列化，通过setInstance()来传入要序列化的数据
    - 确定Function的输入/输出类型：在构建 StreamTransformation时候通过TypeExtractor工具确定Function的输入输出类型。TypeExtractor 类可以根据方法签名、子类信息等蛛丝马迹自动提取或恢复类型信息
    - 确定Function的序列化/反序列化器：构造StreamGraph时，在addOperator()的过程中指定（有点像Hive里Operator对应的Desc）
    - 进行序列化/反序列化的时机：TM管理和调度Task，而Task调用StreamTask，StreamTask中封装了算子的真正处理逻辑。算子处理数据前会收到反序列化封装的数据StreamRecord，并在处理后通过Collector发给下游（构建Collector时已确定SerializationDelegate），序列化的操作交给SerializerDelegate处理

## 第六章 Flink作业执行深度解析
- Flink的四层转换流程：
    1. Program -> StreamGraph
        - 从Source节点开始，每一次transform生成一个StreamNode，两个StreamNode通过StreamEdge连接，Node和Edge一起构成DAG
        - StreamNode有四种transform形式：Source、Flat Map、Window、Sink
    2. StreamGraph -> JobGraph
        - 从Source节点开始，遍历寻找能够嵌到一起的operator，能嵌就嵌，不能嵌的单独生成JobVertex，通过JobEdge连接JobVertex，形成JobVertex层面的DAG
        - 配置Checkpoint在这一步完成，因此需要为每个节点生成byte数组类型的hash值，以获取恢复状态
    3. JobGraph -> ExecutionGraph
        - 从Source节点开始排序，根据JobVertex生成ExecutionJobVertex，根据JobVertex的IntermediateDataSet构建IntermediateResult，并用Result构建上下游依赖关系，形成ExecutionJobVertex层面的DAG，也就是ExecutionGraph
    4. ExecutionGraph -> 物理执行计划
        - 没什么好说的
- **Flink1.5之后新加的Dispatcher可以通过按需分配Container的方式允许不同算子使用不同container配置，这里我暂时想不到具体实现方法，日后回来填坑**

## 第七章 网络流控及反压剖析
- 为什么需要流控：Producer与Consumer吞吐量不一致，上游快的话会导致下游数据积压，如果缓冲区有限则新数据丢失，如果缓冲区无界内存会爆掉
- 网络流控的实现：
    - 静态限速：可以在Producer端实现类似Rate Limiter这样的静态限流，但局限性在于事先无法预估Consumer的承受速率，并且这个通常会动态地波动
    - 动态反馈/自动反压：Consumer及时给Producer做feedback，告知自己能承受的速率，动态反馈分两种：
        - 负反馈：接受速率小于发生速率时告知Producer降低发送率
        - 正反馈：发送速率小于接收速率时告知Producer提高发送率
- Flink做网络传输时数据的流向为Flink->Netty->Socket三级每级都有一个缓冲区，接收端也一样
- Flink1.5之前直接通过TCP自带的流控实现feedback：通过ACK指定消息的offset，通过window指定消息大小（基于本地缓冲区内的消息处理情况，本地缓冲区就是一个滑动窗口）
- TCP有一个ZeroWindowProbe机制可以防止当窗口大小为0时上游无法感知下游重生的数据请求。发送端会定期发送1字节的探测消息，接收端会把窗口大小进行反馈
- ExecutionGraph中的Intermediate Result Partition用于上游发送数据，Input Gate用于下游读取数据
- Flink中的反压传播需要考虑两种情况：
    1. 跨TaskManager：反压如何从下游Input Gate传到上游ResultPartition
    2. TaskManager内：反压如何从Result Partition传播到Input Gate
- TaskManager中会有一个统一的Network BufferPool被所有Task共享，在初始化时会从堆外内存中申请内存（无需依赖GC释放空间）。之后可以为每个ResultSubPartition创建Local BufferPool
- 对于跨TM的反压：
    - 当Input Gate使用的Local BufferPool用完时，会向Network BufferPool申请新空间（有上限），再用完时会直接禁掉Netty Autoread
    - Netty停止从Socket读取数据后，Socket的缓冲区很快也满了
    - Socket会把Window=0发送给发送端（TCP的滑动窗口机制）
    - 发送端Socket停止发送之后，它的Buffer也很快就会堆满
    - Netty检测到Socket无法写之后，会停止向Socket写数据
    - Netty停止写后，很快自己的Buffer会水涨船高，但它的Buffer是无界的，可以通过Netty的水位机制中high watermark来控制上界，超过后会将其channel置为不可写
    - ResultSubPartition在写之前会检测Netty是否可写，不可写就不写了
    - 压力这时给到ResultSubPartition，它也会向Local BufferPool和Network BufferPool申请内存
    - 发送端的Local BufferPool达到上限后整个Operator都会停止写数据，反压目的达成
- 对于TM内的反压：
    - ResultSubPartition无法继续写入数据后，Record Writer的写也会被阻塞
    - Operator的输入和输出在同一线程执行，Writer阻塞了，Reader也会停止从InputChannel读数据
    - 上游还在不停地发，最终这个TM的Input Gate的Buffer会被耗尽，就变成前面那种情况了
- Flink1.5之后使用Credit-based的反压策略
- 基于TCP的反压策略弊端在于堵塞的链条太长，生效延迟也大，并且单个task导致的反压会阻断整个TM的socket，连checkpoint barrier都发不出去了
- 所谓的Credit-based就是在Flink的层面实现类似TCP流控的反压机制，类似window
- Credit-based的反压机制：
    - 每次ResultSubPartition向InputChannel发送消息时都会发送一个backlog size，告诉下游准备发送多少消息
    - 下游会计算自己有多少Buffer可以用于接收消息，如果充足，就返还给上游一个Credit，表示准备好了（底层还是用Netty和Socket通信），如果不足就去申请Local BufferPool，直到达到上限，则返回Credit=0
    - ResultSubPartition接收到Credit=0后就不再向Netty传输数据，上游TM的Buffer也会很快耗尽，这样就不用从Socket到Netty这样一层层向上反馈，降低了延迟，并且Socket也不会被堵
- 某些场景仍然需要静态反压，比如有的外部存储（ES）无法把反压传播给Sink端，就需要用静态限速的方式在Source端做限流

## 第八章 详解Metrics原理与实战 暂时用不上  
## 第九章 Flink Connector开发
- Flink有四种数据读写方式
    1. Flink预定义的Source和Sink
        - 基于文件的Source/Sink
        - 基于Socket的Source/Sink
        - 基于Collections、Iterators的Source；标准输出/标准错误
    2. Flink自带的Boundled connectors
        - 例如Kafka source与sink，Es sink等
        - 在二进制发布包中没有，提交job的jar包记得要把相关connector打进去
    3. 第三方Apache Bahir项目中提供的连接器
        - 从Apache Spark中独立出来的项目
        - 提供flume、redis连接器（sink）
    4. 通过异步IO
        - 常见场景是关联MySQL中某个表
- Flink Kafka Connector
    - 针对不同kafka版本提供了不同的consumer/producer
- Flink Kafka Consumer
    - 提供了反序列化（数据在kafka中是二进制字节数组）的schema类
    - 封装了消费offset位置设置的api
    - 支持topic和partition动态发现（使用独立线程定期获取kafka元数据，并且topic用正则表达式配置）
    - commit offset的方式：如果checkpoint关闭，则依赖于kafka客户端的auto-commit机制。如果checkpoint打开，则Flink在自己的State里管理消费的offset，此时提交kafka仅作为外部监视消费进度（通过setCommitOffsetsOnCheckpoints()来设置当checkpoint成功提交时提交offset到kafka）
    - Timestamp Extraction/Watermark生成：每个partition一个assigner，watermark为多个partition对齐后值（在source前而不是后对齐，以避免出现数据丢失）
- Flink Kafka Producer
    - Producer分区：默认用（taskID%partition个数）来分区，此时若sink数量少于partition，会有partition没人写入。这时将partitioner设置为null就会改用round robin，且数据有key时会做分区散列
    - 容错：使用参数setFlushOnCheckpoint，控制在checkpoint时flush数据到kafka，保证数据已写入到kafka，能达到at-least-once语义。否则缓存在kafka的数据有可能丢（Flink kafka 011版本通过两阶段提交的sink结合kafka事务可以做到端到端Exactly-once）

## 第十章 Flink State最佳实践
- 再来讲讲两种state的区别：
    1. 是否存在当前处理的key（current key）：operator state没有，keyed state数值总与一个Current key对应
    2. 存储对象是否on heap：oeprator state backend仅有on-heap实现，keyed state backend有on/off heap（RocksDB）
    3. 是否需要手动声明snapshot和restore方法：operator state需要手动实现这俩方法；keyed state由backend自行实现，对用户透明
    4. 数据大小：operator state数据规模较小；keyed state相对较大（这个结论是经验判断，不是绝对判断）
- 一般生产中会选择FsStateBackEnd或RocksDBStateBackEnd，前者性能更好，日常存储于堆内存，有OOM风险，不支持增量checkpoint；后者无需担心OOM，大部分时候选这个
- 正确地清空当前state：state.clear()只能清除当前key对应的value，如果要清空整个state，需要借助applyToAllKeys方法。如果只是想清除过期state，用state TTL功能的性能会更好

## 第十一章 TensorFlow On Flink 我暂时用不上
## 第十二章 深度探索Flink SQL
- Blink Planner中SQL/Table API转化为Job Graph的流程：
    1. SQL/Table API解析验证：SQL会被解析成SqlNode Tree抽象语法树，然后做验证，会访问FunctionManager（查询用户定义的UDF以及是否合法）和CatalogManager（检查Table或Database是否存在），验证通过后会生成一个Operation DAG
    2. 生成RelNode：Operation DAG会被转化为RelNode（关系表达式） DAG
    3. 优化：优化器基于规则和统计信息对RelNode做各种优化，Blink Planner中绝大部分的优化规则都是Stream Batch共享的。但差异在于Batch没有状态的概念，而Stream不支持sort，所以Blink Planner还是定义了两套规则集以及对应的Physical Rel：BatchPhysicalRel和StreamPhysicalRel。优化完之后生成的是PhysicalRelDAG
    4. 转化：PhysicalRelDAG会继续转化为ExecNode，属于执行层（Blink）的概念。ExecNode中会进行大量的CodeGen操作，还有非Code的Operator操作，最终生成Transformation DAG
    5. 生成可执行Job Graph：Transformation DAG最终被转化成Job Graph，完成了整个解析过程
- 批处理模式下reuse-source优化可能会导致死锁，具体场景是在执行 hash-join或者nested-loop-join时一定是先读build端再读probe端，如果启用reuse-source-enabled，当数据源是同一个Source时，Source的数据会同时发送给build和probe端。这时build端的数据将不会被消费，导致join操作无法完成，整个join就被卡住（Stream模式不会有死锁，因为不涉及join选边）
- 为了解决死锁问题，Blink Planner会先将probe端数据落盘（因此会多一次磁盘写操作，需要权衡），这样build端读数据的操作才会正常，等build端数据全部读完之后，再从磁盘中拉取probe端的数据，从而解决死锁（**这不就有点像我做的那个动态分区裁剪功能？**）
- 这一部分讲的很多SQL执行层面的优化思路其实和hive是有一定共通之处的，尤其是批处理模式下的优化规则。但流模式下一旦数据源从有界变为无界之后，看待问题（比如聚合的优化）的方式需要做一些改变，这个是日后需要关注的部分

## 第十三章 Python API应用实践 我暂时用不上