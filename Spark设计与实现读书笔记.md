# 第一部分 大数据处理框架的基础知识
## 第1章 大数据处理框架概览
### 1.4 大数据处理框架的四层结构
- 一个大数据应用可以表示为<输入数据，用户代码，配置参数>。应用的输入数据一般以分块（如128MB）的形式预先存储在分布式文件系统（如HDFS）上。用户在向大数据处理框架提交应用之前，需要指定数据存储位置，撰写数据处理代码，并设定配置参数。之后用户将应用提交给大数据处理框架运行。
- 大数据处理框架大体可以分为四层结构：用户层、分布式数据并行处理层、资源管理与任务调度层、物理执行层。
- 以Spark为例，在用户层中，用户需要准备数据、开发用户代码、配置参数。之后，分布式数据并行处理层根据用户代码和配置参数，将用户代码转化成逻辑处理流程（数据单元及数据依赖关系），然后将逻辑处理流程转化为物理执行计划（执行阶段及执行任务）。资源管理与任务调度层根据用户提供的资源需求来分配资源容器，并将任务（taks）调度到合适的资源容器上运行。物理执行层实际运行具体的数据处理任务。
#### 1.4.1 用户层
- 如前所述，将一个大数据应用表示为<输入数据，用户代码，配置参数>
1. 输入数据
    - 对于批式大数据处理框架，如Hadoop、Spark，一般以分块的形式预先存储，可以存在分布式文件系统如HDFS或分布式KV数据库如HBase上，也可以存放到关系数据库中。输入数据在应用提交后会由框架进行自动分块，每个分块一般对应一个具体执行任务（task）。
    - 对于流式大数据处理框架，如Spark Streaming和Flink，输入数据可以来自网络流（socket）、消息队列（kafka）等。数据以微批（多条数据形成一个微批，称为mini-batch）或者连续（一条接一条，称为continuous）的形式进入流式大数据处理框架。
    - 对于大数据应用，数据的高效读取常常成为影响系统整体性能的重要因素。最直观的优化方式就是降低磁盘I/O。如PACMan根据一定策略提前将task所需部分数据缓存到内存中，以提高task的执行性能。Tachyon（Alluxio）构造了一个基于内存的分布式数据存储系统，用户可以将不同应用产生的中间数据缓存到Alluxio中，而不是直接缓存到框架中，可以加速中间数据的写入和读取，以及框架的内存消耗，以实现加速不同的大数据应用（如Hadoop、Spark等）之间的数据传递和共享
2. 用户代码
    - 可以是用户写手的MR代码，或基于其他大数据处理框架的具体应用处理流程的代码。
    - MR提供的map和reduce函数的处理逻辑比较固定单一，难以支持复杂数据操作，比如常见的排序操作sort和数据库表的关联操作join等。因此Dryad和Spark提供了更加通用的数据操作符，如flatMap等。
    - 在实际系统中，用户撰写用户代码后，大数据处理框架会生成一个Driver程序，将用户代码提交给集群运行。例如在Hadoop MapReduce中，Driver程序负责设定输入、输出数据类型，并向MR框架提交作业；在Spark中，Driver程序不仅可以产生数据、广播数据给各个task，还可以收集task的运行结果，最后在Driver程序的内存中计算出最终结果。
    - 除了手写底层操作代码，用户还可以利用高层语言或高层库来间接产生用户代码，通过这种方式生成的代码是二进制的，map和reduce等函数代码不可见。
    - 一些高层库还提供了更简单的方式生成用户代码，如使用Spark的机器学习库MLlib时，用户只需要选择算法和设置算法参数，MLlib即可自动生成可执行的Spark作业了。
3. 配置参数
    - 一个大数据应用可以有很多配置参数，如Hadoop支持200多个配置参数。这些配置参数可以分为两大类：
    1. 与资源相关的配置：例如buffer size定义框架缓冲区的大小，影响map/reduce任务的内存用量。在Hadoop中，map/reduce任务实际启动一个JVM来运行，因此用户还要设置JVM的大小，也就是heap size。在Spark中，map/reduce任务在资源容器（Executor JVM）中以线程的方式执行，用户需要估算应用的资源需求量，并设置应用需要的资源容器个数、CPU个数和内存大小。
    2. 与数据流相关的配置：例如，Hadoop和Spark中者可以设置partition函数、partition个数和数据分块大小。partition函数定义如何划分map的输出数据。partition个数定义产生多少个数据块，也就是有多少个reduce任务会被运行。数据分块大小定义map任务的输入数据大小。
    - Hadoop/Spark框架本身没有提供自动优化配置参数的功能，因此工业界和学术界研究了如何通过寻找最优配置参数来对应用进行性能调优。几个例子：
    1. StarFish的Just-In-Time优化器，可以对Hadoop应用的历史运行信息进行分析，并根据分析结果来预测应用在不同配置参数下的执行时间，以选择最优参数。
    2. Verma等讨论了在给定应用完成时限的情况下，如何为Hadoop应用分配最佳的资源（map/reduce slot）来保证应用能够在给定时限内完成。
    3. DynMR通过调整任务启动时间、启动顺序、任务个数来减少任务等待时间和由于过早启动而引起的任务之间资源竞争。
    4. MROnline根据任务执行状态，使用爬山法寻找最优的缓冲区大小和任务内存大小，以减少应用执行时间。
    5. Xu等研究了如何离线估计MapReduce应用内存用量，即先用小样本数据运行应用，然后根据应用运行信息来估算应用在大数据上的实际内存消耗。
    6. SkewTune可以根据用户自定义的代码函数来优化数据划分算法，在保持数据输入顺序的同时，减少数据倾斜的问题。
#### 1.4.2 分布式数据并行处理层
- 分布式数据并行处理层首先将用户提交的应用转化为较小的应用任务，然后通过调用底层的资源管理与任务调度层实现并行执行。
- 在Hadoop MapReduce中，这个转化过程是直接的，因为MR具有固定的执行流程：map-shuffle-reduce。map和reduce阶段各自包含多个可以并行执行的任务。map负责将输入的分块数据进行map处理，并将其输出结果写入缓冲区，然后对缓冲区中的数据进行分区、排序、聚合等操作，最后溢写到磁盘上的不同分区中。reduce则首先将map任务输出的对应分区数据通过网络传输拷贝到本地内存中，内存空间不够时，会将内存数据排序后写入磁盘，然后经过归并、排序等阶段产生reduce的输入数据。reduce处理完输入数据后，将输出数据写入分布式文件系统中。
- 在Spark中应用的转化过程包含两层：逻辑处理流程、执行阶段与执行任务划分。
- Spark首先根据用户代码中的数据操作语义和操作顺序，将代码转化为逻辑处理流程。逻辑处理流程包含多个数据单元和数据依赖，每个数据单元包含多个数据分块。然后，框架对逻辑处理流程进行划分，生成物理执行计划。该计划包含多个执行阶段（stage），每个执行阶段包含若干执行任务（task）。
- 为了将用户代码转化为逻辑处理流程，Spark对输入/输出、中间数据进行了更具体的抽象处理，将这些数据用一个统一的数据结构表示，即RDD（Resilient Distributed Datasets，弹性分布式数据集）。
- 在RDD上可以执行多种数据操作，如简单的map以及复杂的cogroup、join等。
- 一个RDD可以包含多个数据分区（partition），parentRDD和childRDD之间通过数据依赖关系关联，支持一对一和多对一等数据依赖关系。数据依赖关系的类型由数据操作的类型决定。
- 为了将逻辑处理流程转化为物理执行阶段，Spark首先根据RDD之间的数据依赖关系，将整个流程划分为多个小的执行阶段（stage）。之后，在每个执行阶段形成计算任务（task），计算任务的个数一般与RDD中分区的个数一致。
- 与MR不同的是，一个Spark job可以包含很多个执行阶段，而且每个执行阶段可以包含多种计算任务，因此并不能严格地区分每个执行阶段中的任务是map任务还是reduce任务。
- 在Spark中，用户可以通过调用cache接口使框架缓存可被重用的中间数据。例如，当前job的输出可能会被下一个job用到，那么用户可以使用cache()来对这些数据进行缓存。
#### 1.4.3 资源管理与任务调度层
- 从系统架构上讲，大数据处理框架一般是主-从结构（Master-Worker）。主节点负责接收用户提交的应用，处理请求，管理应用运行的整个生命周期。从节点负责执行具体的数据处理任务，并在运行过程中向主节点汇报任务的执行状态。比如在Hadoop MapReduce中，在主节点运行的JobTracker进程首先接收用户提交的job，然后根据job的输入数据和配置等信息将job分解为具体的数据处理任务，然后将task交给任务调度器调度运行。任务调度器根据各个从节点的资源总量与资源使用情况将map/reduce task分发到合适的从节点的TaskTracker中。TaskTracker进程会为每个task启动一个进程执行task的处理步骤。每个从节点可以同时运行的task数目由该节点的CPU个数等资源状况决定。
- 大数据处理服务器集群一般由多个用户共享，当集群资源充足的情况下，集群会同时运行多个job，每个job包含多个map/reduce task。同一个节点上运行的task可以属于不同的job。
- Spark运行不同的部署模式，如Standalone、YARN和Mesos模式。其中Standalone模式与MR部署模式基本类似，唯一区别是MR部署模式为每个task启动一个JVM进程运行，而且是在task将要运行时启动JVM，而Spark是预先启动资源容器（Executor JVM），然后当需要执行task时，再在容器里启动task线程运行。
- 在运行大数据应用前，框架还需要对应用job及其任务task进行调度，主要目的是通过设置不同的策略来决定应用或任务获得资源的先后顺序。典型的调度方式有FIFO和Fair等。
- 调度器有两种类型：应用调度器（决定多个应用app执行的先后顺序）和任务调度器（决定多个任务task的执行先后顺序）。
#### 1.4.4 物理执行层
- 物理执行层负责启动task，执行每个task的数据处理步骤。
- 不像MR的map、shuffle、reduce，Spark中一个应用可以有更多的执行阶段stage，如迭代型应用可能有几十个执行阶段，每个执行阶段也包含多个task。这些执行阶段可以形成复杂的DAG图结构，在物理执行时首先执行上游stage中的task，完成后执行下游stage中的task。
- MR中每个task对应一个进程，以JVM的方式来运行，因此task的内存用量就是JVM的堆内存用量。
- Spark中每个task对应JVM中的一个线程，一个JVM可能同时运行了多个task。在应用未运行前，难以预知task的内存消耗和执行时间，以JVM的堆内存用量。
- 从应用特点分析，可以将task执行过程中主要消耗内存的数据分为以下3类：
    1. 框架执行时的中间数据：例如map输出到缓冲区的数据和reduce在shuffle阶段暂存到内存中的数据。
    2. 框架缓存数据：例如在Spark中，用户调用cache接口缓存到内存中的数据。
    3. 用户代码产生的中间计算结果：例如用户代码调用map、reduce、combine，在处理输入数据时会在内存中产生中间计算结果。
- Spark框架是基于内存计算的，它将大量输入数据和中间数据缓存到了内存中，有效提高交互型job和迭代型job的执行效率。
- 由于大数据应用的内存消耗量很大，当前许多研究关注如何改进大数据处理框架的内存管理机制，以减少应用内存消耗。

## 第2章 Spark系统部署与应用运行的基本流程
### 2.2 Spark系统架构
- Spark application：即Spark应用，指的是1个可运行的Spark程序，该程序包含main函数，其数据处理流程一般先从数据源读取数据，再处理数据，最后输出结果。同时，应用程序也包含了一些配置参数，如需要占用的CPU个数，Executor内存大小等。用户可以使用Spark本身提供的数据操作来实现程序，也可以通过其它框架（如Spark SQL）来实现应用，Spark SQL可以将SQL语句转化成Spark程序执行。
- Spark Driver：即Spark驱动程序，指实际在运行Spark应用中main函数的进程。Driver一般位于Master节点上，但独立于Master进程。如果是YARN集群，Driver也可能被调度到Worker节点上运行。也可以在自己的PC上运行Driver，通过网络与远程的Master进程连接，但一般不推荐这么做，因为不但需要本地安装一个与集群一样的Spark版本，而且自己的PC一般和集群不在一个网段，Driver和Worker节点之间的通信会很慢。
- Executor：即Spark执行器，是Spark计算资源的一个单位。Spark先以Executor为单位占用集群资源，然后可以将具体的计算任务分配给Executor执行。由于Spark是由Scala编写的，Executor在物理上是一个JVM进程，可以运行多个线程（任务）。
- task：即Spark应用的计算任务。Driver在运行Spark应用的main函数时，会将应用拆分成多个计算任务，然后分配给多个Executor执行。task是Spark中最小的计算单位，不能再拆分。task以线程方式运行在Executor进程中，执行具体的计算任务，如map算子、reduce算子等。由于Executor可以配置多个CPU，而1个task一般使用1个CPU，因此当Executor具有多个CPU时，可以运行多个task。Executor的总内存大小由用户配置，其由多个task共享。
- Hadoop MR中的每个task以一个JVM进程的方式运行，好处是可以让task之间相互独立，每个task独享进程资源，不会相互干扰，而且监控管理比较方便，但坏处是task之间不方便共享数据（需要将共享数据加载到每个task进程中，造成重复加载和内存资源浪费），并且在应用执行过程中需要不断启停新旧task，进程的启动和停止需要做很多初始化工作，会降低执行效率。
- Spark中的每个task以JVM中的一个线程的方式运行，好处是数据共享和执行效率得到提高，坏处是线程间会有资源竞争，而且Executor JVM的日志会包含多个并行task的日志，较为混乱。
- 每个Worker进程中存在一个或者多个ExecutorRunner对象，每个对象管理一个Executor。Executor持有一个线程池，每个线程执行一个task。
- Worker进程通过持有一个或多个ExecutorRunner对象来控制各自对应的CoarseGrainedExecutorBackend进程的启停（Executor位于其中）
- 每个Spark应用启动一个Driver和多个Executor，每个Executor里面运行的task都属于同一个Spark应用。
### 2.3 Spark应用例子
### 2.3.1 用户代码基本逻辑
- 一般不需要在编写应用时指定map task的个数，因为其可以通过“输入数据的大小/每个分片大小”来决定，而reduce task的个数一般在使用算子时通过设置partition number来间接设置。
- Spark编程与使用普通语言编写数据处理程序的不同：
    - 使用普通语言编程：处理的数据在本地，程序也在本地进程中运行，可以随意定义变量、函数、控制流（分支、循环）等，编程灵活、受限较少，且程序按照既定顺序执行、输出结果。
    - 使用Spark编程：首先要声明SparkSession的环境变量才能够使用Spark提供的数据操作，然后使用Spark操作来定义数据处理流程。此时只是定义了数据处理流程，而并没有让Spark真正开始计算，就像在一个画布上画出了数据处理流程，包括哪些数据处理步骤以及这些步骤如何连接，每步的输入和输出是什么。至于这些步骤和操作如何在系统中并行执行，用户并不需要关心。有点像SQL的执行。
- 在Spark中，唯一需要注意声明的数据处理流程在使用action()操作时，Spark才真正执行处理流程，如果整个程序中没有action操作，就不会执行数据处理流程。而在普通程序中程序一步步按照顺序执行，无此限制。
### 2.3.2 逻辑处理流程
- 先建立DAG型的逻辑处理流程(Logical plan)，然后根据逻辑处理流程生成物理执行计划(Physical plan)，后者包含具体的计算任务task，最后Spark将task分配到多台机器上执行。
### 2.3.3 物理执行计划
- Spark根据数据依赖关系，将逻辑处理流程转化为物理执行计划，包括执行阶段stage和执行任务task。具体包括下面三个步骤：
    1. 确定应用会产生哪些作业（job），一般情况下对应action操作的个数。（有时一个action可以被优化为多个作业，如某些情况下的saveAsTextFile，或有时没有明显action也会启动作业，如foreach等操作）
    2. 根据逻辑处理流程中的数据依赖关系，将每个job的处理流程拆分为执行阶段stage。如果两个RDD是一对一的关系则可以放在一起处理形成一个stage。多对多的RDD则会被分别处理形成两个stage。
    3. 对于每一个stage，根据RDD的分区个数确定执行的task个数和种类。
- 生成task后，task可以被调度到Executor上执行，在同一个stage中的task可以并行执行。
- 拆分stage的好处：
    1. stage中生成的task不会太大，也不会太小，而且是同构的，便于并行执行。
    2. 可以将多个操作放在一个task里处理，使得操作可以进行串行、流水线式的处理，提高数据处理效率。
    3. stage可以方便错误容忍，比如一个stage失效时可以重新运行这个stage，而不是整个job。
### 2.3.4 可视化执行过程
- 可以根据Spark提供的执行界面，即Job UI来分析一个Spark应用的逻辑处理流程和物理执行计划。
- 可以根据stage的task个数来判断RDD的分区个数。

# 第二部分 Spark大数据处理框架的核心理论
## 第3章 Spark逻辑处理流程
### 3.1 Spark逻辑处理流程概览
- 典型的逻辑处理流程主要包含四部分：
    1. 数据源：
        - 表示的是原始数据，可以存放在本地文件系统和分布式文件系统中，或网络流中。
    2. 数据模型：
        - 使用普通的面向对象程序时，数据会被抽象为内存中的对象(Object)。
        - Hadoop的MapReduce构架将输入/输出、中间数据抽象为<K,V> record的形式，这种方式的优点是简单易操作，缺点是过于细粒度。
        - 由于<K,V> record没有进行更高层的抽象，导致只能使用map(K,V)的固定形式去处理数据，而无法使用面向对象程序的灵活数据处理方式，如records.operation()的方式。
        - Spark针对这个缺点，将输入/输出、中间数据抽象表示为统一的数据模型（数据结构），命名为RDD。每个输入/输出、中间数据可以是一个具体的实例化的RDD，其中可以包含各种类型的数据，比如普通的Int、Double，或<K,V> record等。
        - RDD与普通数据结构的主要区别有两点：
            1. RDD只是一个逻辑概念，在内存中并不会真正地为某个RDD分配存储空间（除非其需要被缓存）。RDD中的数据只会在计算中产生，并在计算完成后消失。
            2. RDD可以包含多个数据分区，不同数据分区可以由不同的任务在不同节点进行处理。
    3. 数据操作：
        - 定义了数据模型后，可以对RDD进行各种数据操作，Spark将这些数据操作分为两种：transformation()和action()。两者的区别是后者一般是对数据结果进行后处理(post-processing)，产生输出结果，并触发Spark提交job真正执行数据处理任务。
        - transformation一词隐含了单向操作的意思，也就是rdd1使用transformation()之后会生成新的rdd2，而不会对rdd1本身进行修改。这点和普通面向对象程序中的对象不同。
        - 在Spark中，因为数据操作一般是单向操作，通过流水线执行，还需要进行错误容忍等，所以被设计成一个不可变类型。
    4. 计算结果处理：
        - 由于RDD实际上是分布在不同机器上的，所以大数据应用的结果计算分为两种方式：一种是直接将结果放进HDFS中，这种方式一般不需要在Driver端进行集中运算；另一种方式则是需要在Driver端进行集中运算，如统计RDD中的元素数目，需要先使用多个task统计每个RDD中分区(partition)的元素数目，再将它们汇集到Driver端进行加和计算。
### 3.2 Spark逻辑处理流程生成方法
- Spark实际生成的逻辑处理流程图往往比头脑中直观的想象更加复杂，例如会多出几个RDD，每个RDD会有不同的分区个数，RDD之间的数据依赖关系不同，等等。
- 将应用程序自动转化为确定性的逻辑处理流程，需要解决以下3个问题：
    1. 根据应用程序如何产生RDD，产生什么样的RDD？
    2. 如何建立RDD之间的数据依赖关系？
    3. 如何计算RDD中的数据？
#### 3.2.1 根据应用程序如何产生RDD，产生什么样的RDD
- 一种简单解决方法是对程序中每一个数据进行操作，也就是用transformation()方法返回一个新的RDD。这种方法的主要问题是只适用于逻辑比较简单的transformation()，一些复杂的trasformation如join、distinct等，需要对中间数据进行一系列子操作，那么一个Transformation会创建出多个RDD。
- 数据本身可能具有不同的类型，而且是由不同的计算逻辑得到，可能具有不同的依赖关系。因此需要多种类型的RDD来表示这些不同的数据类型、不同的计算逻辑，以及不同的数据依赖。
- Spark实际产生的RDD类型和个数与trasformation的计算逻辑有关。
#### 3.2.2 如何建立RDD之间的数据依赖关系
- 数据依赖关系包括两方面：一方面是RDD之间的依赖关系，如一些transformation会对多个RDD进行操作，则需要建立这些RDD之间的关系。另一方面是RDD本身具有分区特性，需要建立RDD自身分区之间的关联关系。具体地需要解决以下3个问题:
    1. 如何建立RDD之间的数据依赖关系？例如，生成的RDD是依赖于一个parent RDD还是多个parent RDD？
    2. 新生成的RDD应该包含多少个分区？
    3. 新生成的RDD与其parent RDD中的分区间是什么依赖关系？是依赖parent RDD中的一个分区还是多个分区呢？
- 第1个问题可以很自然地解决，对于一元操作，如rdd2 = rdd1.transformation()可以确定rdd2只依赖rdd1。对于二元操作，如rdd3 = rdd1.join(rdd2)，可以确定rdd2同时依赖rdd1和rdd2。二元以上的操作可以类比二元操作。
- 对于第2个问题，在Spark中，新生成的RDD的分区个数由用户和parent RDD共同决定，对于一些transformation()，如join操作，我们可以指定其生成的分区个数，如果个数不指定，则一般取其parent RDD的分区个数最大值。还有一些操作如map，其生成的RDD的分区个数与数据源的分区个数相同。
- 第3个问题比较复杂，分区之间的依赖关系既与transformation的语义有关，也与RDD的分区个数有关。例如在执行rdd2=rdd1.map时，map对rdd1的每个分区中的每个元素进行计算，可以得到新的，类似一一映射，因此不需要改变分区个数。而对于groupByKey之类的聚合操作，在计算时需要对parent RDD中各个分区的元素进行计算，需要改变分区之间的依赖关系，使得RDD中的每个分区依赖其parent RDD中的多个分区。
- Spark设计用于解决第3个问题的通用方法：理论上，分区之间的数据依赖关系可以灵活自定义，如一一映射、多对一映射、多对多映射或者任意映射等。实际上，常见数据操作的数据依赖关系具有一定的规律，Spark将其分为两大类：
1. 窄依赖(NarrowDependency)
    - 官方解释：如果新生成的child RDD中每个分区都依赖parent RDD中的一部分分区，则这个分区依赖关系被称为窄依赖。
    - 窄依赖可以进一步细分为4种依赖：
        1. 一对一依赖(OneToOneDependency)：表示child RDD和parent RDD中的分区个数相同，并存在一一映射关系，比如map和filter等。
        2. 区域依赖(RangeDependency): 表示child RDD和parent RDD的分区经过区域化后存在一一映射关系，比如union等。
        3. 多对一依赖(ManyToManyDependency)：表示child RDD中的一个分区同时依赖多个parent RDD中的分区，比如具有特殊性质的cogroup、join等（下一节讲）。
        4. 多对多依赖(ManyToManyDependency): 表示child RDD中的一个分区依赖parent RDD中的多个分区，同时parent RDD中的一个分区被child RDD中的多个分区依赖，如cartesian。
2. 宽依赖(ShuffleDependency)
    - 官方解释：Represents a dependency on the output of a shuffle stage.
    - 如果从数据流角度解释，宽依赖表示新生成的child RDD中的分区依赖parent RDD中的每个分区的一部分。
    - 窄依赖的多对多依赖中，child RDD的每个分区依赖parent RDD中每个分区的所有部分。而宽依赖中child RDD的每个分区虽然依赖parent RDD中的所有分区，但只依赖这些分区中id为某些值的部分。
- 总的来说，窄依赖和宽依赖的区别是child RDD的各个分区是否完全依赖parent RDD的一个或多个分区。
- 根据数据操作语义和分区个数，Spark可以在生成逻辑处理流程时就明确child RDD是否需要parent RDD的一个或多个分区的全部数据。
- 如果parent RDD的一个或多个分区中的数据全部流入child RDD的某一个分区或者多个分区，则是窄依赖。如果parent RDD分区中的数据需要一部分流入child RDD的某个一个分区，另一部分流入child RDD的另外分区，则是宽依赖。
- 窄依赖在执行时可以在同一个阶段进行流水线操作，不需要进行Shuffle。
- 如何对RDD内部的数据进行分区？Spark采用了三种分区方法：
    1. 水平划分：按照record的索引进行划分。这种方式经常用于输入数据的划分，如先将输入数据上传到HDFS上，HDFS自动对数据进行水平划分，按照文件块(128MB)为单位将输入数据划分为很多个小块，之后每个Spark task可以只处理一个数据块。
    2. Hash划分(HashPartitioner): 使用record的Hash值来对数据进行划分，好处是只需要知道分区个数就能将数据确定性地划分到某个分区中。在水平划分中由于每个RDD中的元素数目和排列顺序不固定，同一个元素在不同RDD中可能被划分到不同的分区。使用Hash划分则不会有这个问题，这种方式经常被用于数据Shuffle阶段。
    3. Range划分(RangePartitioner): 一般适用于排序任务，核心思想是按照元素的大小关系将其划分到不同分区，每个分区表示一个数据区域。假如想对数据进行排序，Range划分会首先将数据按上下界划分为若干份，然后将record分发到相应的分区，最后对每个分区进行内部排序，这个排序过程可以并行执行，排序完成后是全局有序的结果。Range划分需要提前划分好数据区域，需要统计RDD中数据的最大值和最小值，为了简化这个统计过程，Range划分经常采用抽样方法来估算数据区域边界。
#### 3.2.3 如何计算RDD中的数据
- 在确定了数据依赖关系后，相当于知道了child RDD中每个分区的输入数据是什么，那么就只需要使用transformation函数处理这些输入数据，将生成的数据推送到child RDD中对应的分区即可。
- Spark中的大多数trasformation类似数学中的映射函数，具有固定的计算方式（控制流），如map操作需要每读入一个record就进行处理，然后输出一个record。reduceByKey操作对中间结果和下一个record进行聚合计算并输出结果。
- Spark也提供一些类似普通面向对象语言程序的操作，比如mapPartitions可以对分区中的数据进行多次操作后再输出结果。计算逻辑接近Hadoop MapReduce中的map()和cleanup()，对每个到来的<K,V> record都进行处理，等对这些record处理完成后，再对处理结果进行集中输出。
### 3.3 常用的transformation数据操作
- map(func): 使用func对rdd1中的每个record进行处理，输出一个新的record。
- mapValue(func): 对于rdd1中每个<K,V> record，使用func对Value进行处理，得到新的record。
- filter(func): 对rdd1中的每个record进行func操作，如果结果为true，则保留这个record，所有保留的record将形成新的rdd2。
- filterByRange(lower, upper): 对rdd1中的数据进行过滤，只保留[lower, upper]之间的record。
- flatMap(func): 对rdd1中每个元素（如list）执行func操作，得到新元素，然后将所有新元素组合得到rdd2。主要适用于rdd1中是一个集合的元素。
- flatMapValues(func): 与flatMap()相同，只针对record中的Value进行func操作。
- flatMap()和flatMapValues()操作都会生成一个MapPartitionsRDD，这两个操作生成的数据依赖关系都是OneToOneDependency。
- sample(withReplacement, fraction): 对rdd1中的数据进行抽样，取其中fraction*100%的数据，withReplacement=true表示有放回的抽样，seed表示随机数种子。
- sampleByKey(withReplacement, fractions: Map, seed): 对rdd1中的数据进行抽样，为每个Key设置抽样比例，如Key=1的抽样比例是30%等，withReplacement=true表示有放回的抽样，seed表示随机数种子。
- sample()操作生成一个PartitionwiseSampledRDD，而sampleByKey操作生成一个MapPartitionsRDD，这两个操作生成的数据依赖关系都是OneToOneDependency。sample(false)与sample(true)的区别是前者使用伯努利抽样模型抽样，每个record有fraction*100%的概率被选中；后者使用泊松分布抽样，也就是生成泊松分布，然后按照泊松分布采样，抽样得到的record个数可能大于rdd1中的record个数。
- sampleByKey()可以为每个Key设定被抽取的概率。
- mapPartitions(func): 对rdd1中每个分区进行func操作，输出新的一组数据，其与map()的区别在前一小节中介绍。
- mapPartitionsWithIndex(func): 语义与前一个基本相同，只是分区中的数据带有索引（表示record属于哪个分区）。当程序计算出一个result RDD时，如果想知道这个RDD中包含多少个分区，以及每个分区中包含哪些record，就可以使用mapPartitionsWithIndex()来输出这些数据。
- mapPartitions和mapPartitionsWithIndex操作更像是过程式编程，给定一组数据后，可以使用数据结构持有中间处理结果，也可输出任意大小、任意类型的一组数据。这两个操作还可以用来实现数据库操作，比如在mapPartitions()中先建立数据库连接，然后将每一个新来的数据iter.next()转化成数据表中的一行，并将其插入数据库中。map()就不能这么操作，因为它会对每个record执行同样的操作，这样每个record都会建立一个数据库连接，造成数据库重复连接。
- partitionBy(partitioner): 使用新的partitioner对rdd1进行重新分区，partitioner可以是HashPartitioner、RangePartitioner等，要求rdd1是<K,V>类型。
- groupByKey([numPartitions]): 将rdd1中的<K,V> record按照Key聚合在一起，形成<K, list(V)>（实际是<K, CompactBuffer(V)>），numPartitions表示生成的rdd2的分区个数。
- groupByKey类似SQL语言中的GroupBy算子，不同的是groupByKey是并行执行的。与前面介绍的只包含窄依赖的transformation不同，groupByKey引入了宽依赖，可以对child RDD的数据进行重新分区组合，因此groupByKey输出的parent RDD的分区个数更加灵活，分区个数可以由用户指定，如果用户没有指定就默认为parent RDD中的分区个数。缺点是在Shuffle时会产生大量的中间数据、占用内存大，多数情况下会选用下面介绍的reduceByKey。
- reduceByKey(func, [numPartitions]): 与groupByKey()类似，也是将rdd1中的具有相同Key的record聚合在一起，不同的是在聚合的过程中使用func对这些record的Value进行融合计算。与groupByKey只在宽依赖后按Key对数据进行聚合不同，reduceByKey实际包括两步聚合。第一步在宽依赖之前对RDD中的每个分区中的数据进行一个本地化的combine聚合操作，也称为mini-reduce或map端combine。首先对ParallelCollectionsRDD中的每个分区进行combine操作，将具有相同Key的Value聚合在一起，并利用func进行reduce聚合操作，这一步由Spark自动完成，并不形成新的RDD。第2步，reduceByKey()生成新的ShufledRDD，将来自rdd1中不同分区且具有相同Key的数据聚合在一起，同样利用func进行reduce聚合操作。其中combine和reduce的计算逻辑采用同一个func。需要注意的是func需要满足交换律和结合律，因为Shuffle并不保证数据的到达顺序。并且由于宽依赖需要对Key进行Hash划分，Key不能是特别复杂的类型，比如Array。
- reduceByKey虽然类似Hadoop MR中的reduce函数，但灵活性没后者好，因为后者的reduce函数输入的是<Key, list(Value)>，可以在对list进行任意处理后输出<Key, new_Value>。而前者中的func有限制，即只能对record一个接一个连续处理、中间计算结果也必须与Value同一类型、必须满足交换律和结合律。
- 在性能上，reduceByKey可以在Shuffle之前使用func对数据进行聚合，减少了数据传输量和内存用量，效率比groupByKey高。
- aggregateByKey(zeroValue, seqOp, combOp, [numPartitions])：是一个通用的聚合操作，可以看作一个更一般的reduceByKey()。相比于reduceByKey，aggregateByKey可以把combine和reduce两个函数的计算逻辑分开。另外，有时候进行reduce操作时需要一个初始值，而reduceByKey没有初始值，因此aggregateByKey还提供了一个zeroValue参数，来为seqOp提供初始值。这个transformation在Spark应用中的使用频率很高，如Spark MLlib中。
- 在reduceByKey中，func要求参与聚合的record和输出结果是同一个类型，而aggregateByKey中zeroValue和record可以是不同类型，但seqOp的输出结果与zeroValue是同一类型的，在一定程度上提高了灵活性。
- reduceByKey可以看作特殊版的aggregateByKey。当seqOp处理的中间数据量很大，出现Shuffle spill的时候，Spark会在map端执行combOp()，将磁盘上经过seqOp处理的<K,V> record与内存中经过seqOp处理的<K,V> record进行融合。reduceByKey可以看作seqOp=combOp=func版本的aggregateByKey()。
- combineByKey(createCombiner, mergeValue, mergeCombiners, [numPartitions]): 是一个通用的基础聚合操作，常用的聚合操作如aggregateByKey和reducebyKey都是利用combineByKey实现的。createCombiner是一个函数，可以提供比zeroValue更强大的功能，比如根据每个record的value值提供不同的初始值。
- foldByKey(zeroValue, numPartitions, func): 是一个简化的aggregateByKey，seqOp和combineOp共用一个func。基于aggregateByKey实现，功能介于其与reduceByKey之间，可以粗略看作多了初始值的reduceByKey。
- cogroup/groupWith(otherDataset, [numPartitions]): 将多个RDD中具有相同Key的Value聚合在一起，假设rdd1包含<K,V> record，rdd2包含<K,W> record，则两者聚合结果为<K,list(V), list(W)>。这个操作还有另一个名字：groupWith
- cogroup与groupByKey的不同在于cogroup可以将多个RDD聚合为一个RDD。因此其生成的RDD与多个parent RDD存在依赖关系。一般来说聚合关系需要宽依赖，但也存在特殊情况。比如在groupByKey中如果child RDD和parent RDD使用的partitioner相同且分区个数相同，就没必要使用宽依赖，直接一对一窄依赖即可。更为特殊的是，由于cogroup可以聚合多个RDd，因此可能对一部分RDD采用宽依赖，而对另一部分RDD采用一对一窄依赖。
- Spark在决定RDD之间的数据依赖时除了考虑transformation的计算逻辑，还考虑child RDD和parent RDD的分区信息，当分区个数和partitioner都一致时，说明parent RDD中的数据可以直接流入child RDD，不需要shuffle，这样可以避免数据传输，提高执行效率。
- cogroup最多支持4个RDD同时进行cogroup。cogroup实际生成两个RDD：CoGroupedRDD将数据聚合在一起，MapPartitionsRDD将数据类型转变为CompactBuffer（类似Java的ArrayList）。当cogroup聚合的RDD包含很多数据时，Shuffle这些中间数据会增加网络传输，而且需要很大内存来存储聚合后的数据，效率较低。
- join(otherDataset, [numPartitions]): 将两个RDD中的数据关联在一起，与SQL中的join类似。假设rdd1中的数据为<K,V> record，rdd2中的数据为<K,W> record，那么join之后的结果为<K,(V,W)> record。与SQL中的算子类似，join还有其他形式，如leftOuterJoin, rightOuterJoin, fullOuterJoin等。
- join操作实际上建立在cogroup之上，首先利用CoGroupedRDD将具有相同Key的Value聚合在一起，形成<K, [list(V), list(W)]>，然后对其进行笛卡尔积计算并输出结果<K, (V,W)>，其中list表示CompactBuffer。在实际实现中，join首先调用cogroup生成CoGroupedRDD和MapPartitionsRDD，然后计算MapPartitionsRDD中[list(V), list(W)]的笛卡尔积，生成MapPartitionsRDD。
- cartesian(otherDataset): 计算两个RDD的笛卡尔积，若rdd1有m个分区，rdd2有n个分区，则此操作会生成m*n个分区，输出rdd1中m个分区与rdd2中的n个分区两两组合后的结果，每个结果形成一个分区，结果分区中的元素是rdd1和rdd2中对应分区的元素的笛卡尔积。此操作形成的数据依赖关系虽然比较复杂，但归属于多对多的窄依赖。
- sortByKey([ascending], [numPartitions]): 对rdd1中<K,V> record进行排序，只按照Key进行排序，在相同Key的情况下并不对Value进行排序。ascending=true时为升序。此操作先将rdd1中不同Key的record分发到ShuffledRDD中的不同分区中，然后在ShuffledRDD的每个分区中按照Key对record进行排序，形成的数据依赖关系为ShuffleDependency。
- sortByKey和groupByKey一样并不需要使用map端的combine。
- 与reduceByKey等操作使用Hash划分来分发数据不同，sortByKey为了保证生成的RDD中的数据是全局有序，采用了Range划分来分发数据，这样可以保证在生成的RDD中，partition1中的所有record的Key小于或大于partition2中所有record的Key。
- 如果需要让Value也有序，可以像Hadoop MR一样把value放入key中形成组合Key，再对这个组合Key定义排序函数来实现，最后用sortByKey只输出key就可以了。或者先用groupByKey将数据聚合成<Key, list(Value)>，然后再使用rdd.mapValues(sort function)操作来对list进行排序。
- coalesce(numPartitions, [shuffle]): 将rdd1的分区个数降低或升高为numPartitions。此操作可以改变RDD的分区个数，而且在不同参数下具有不同的逻辑处理流程。有以下四种情况：
    1. 减少分区个数：会将相邻的分区直接合并在一起，得到rdd2，形成的数据依赖关系是多对一的窄依赖。这种方法的缺点是，当rdd1中不同分区中的数据量差别较大时，直接合并容易造成数据倾斜。
    2. 增加分区个数：并没有什么效果，因为coalesce默认使用窄依赖，不能将一个分区拆分为多份。
    3. 使用Shuffle来减少分区个数：为了解决数据倾斜的问题，可以使用shuffle来减少RDD的分区个数，Spark可以随机将数据打乱，从而使得生成的RDD中每个分区中的数据比较均衡。具体采用的方法是为rdd1中每个record添加一个特殊的key，再根据Key的Hash值将rdd1中的数据分发到rdd2的不同的分区中，然后去掉Key即可。
    4. 使用Shuffle来增加分区个数：通过使用宽依赖，对分区进行拆分和重新组合，解决分区不能增加的问题。
- repartition(numPartitions)：将RDD中的数据进行重新shuffle分区，语义与coalesce(numPartitions, shuffle=true)一致。
- repartitionAndSortWithinPartitions(partitioner)：与repartition()操作类似，将rdd1中的数据重新进行分区，分发到rdd2中。不同的是此操作可以灵活使用各种partitioner，**并且对于rdd2中的每个分区，对其中的数据按照Key进行排序**。这个操作比repartition+sortByKey效率高。
- intersection(otherDataset): 求交集时将rdd1和rdd2中共同的元素抽取出来，形成新的rdd3。核心思想是先利用cogroup将rdd1和rdd2的相同record聚合在一起，然后过滤出在rdd1和rdd2中都存在的record。具体方法是先将rdd1中的record转化为<K, V>，V为固定值null，然后将rdd1和rdd2中的record聚合在一起，过滤掉出现"()"的record（即另一方没有的元素），最后只保留Key，得到交集元素。
- distinct(numPartitions)：去重操作，将rdd1中的数据进行去重，rdd2为去重后的结果。与intersection相似，先将数据转化为<K,V>类型，其中Value为null，然后使用reduceByKey()将这些record聚合在一起，最后使用map只输出Key就可以得到去重后的元素。
- union(otherDataset)：将rdd1和rdd2中的元素合并在一起，得到新的rdd3。形成的数据依赖关系是RangeDependency窄依赖。union形成的逻辑执行流程有两种：
    1. rdd1和rdd2是两个非空的RDD，并且两者的partitioner不一致，且合并后的rdd3为UnionRDD，其分区个数是rdd1和rdd2的分区个数之和，rdd3的每个分区也一一对应rdd1或rdd2中相应的分区。
    2. rdd1和rdd2是两个非空的分区，且两者都使用Hash划分，得到rdd1'和rdd2'。因此rdd1'和rdd2'的partitioner是一致的，都是Hash划分且分区个数相同。它们合并后的rdd3为PartitionerAwareUnionRDD，其分区个数与rdd1'和rdd2'的分区个数相同，且rdd3中的每个分区的数据都是rdd1'和rdd2'对应分区合并后的结果。
- zip(otherDataset)：将rdd1和rdd2中的元素按照一一对应关系（像拉链一样）连接在一起，构成<K,V> record，K来自rdd1，Value来自rdd2。此操作要求rdd1和rdd2的分区个数相同，而且每个分区包含的元素个数相同。生成的RDD名为ZippedPartitionsRDD2，RDD2的意思是对两个RDD进行连接。
- zipPartitions(otherDataset)：将rdd1和rdd2中的分区按照一一对应关系（像拉链一样）连接在一起，形成rdd3。rdd3中的每个分区中的数据为<list(records from rdd1), list(records from rdd2)>，然后可以自定义函数func对这些record进行处理。此操作要求rdd1和rdd2中的分区个数相同，但每个分区包含的元素个数可以不相等。
- zipPartitions首先像拉链一样将rdd1和rdd2中的分区（而非分区中的每个record）按照一一对应关系连接在一起，并提供两个迭代器rdd1Iter和rdd2Iter，来分别迭代每个分区中来自rdd1和rdd2的record。
- zipPartitions可以同时连接2、3或4个rdd，要求参与连接的rdd都包含相同的分区个数。其还有一个参数是preservePartitioning，默认值为false，即生成的rdd继承parent RDD的partitioner，因为继承partitioner可以提升后续操作的执行效率（比如避免Shuffle阶段）。假设rdd1和rdd2的partitioner都为HashPartitioner，那么preservePartitioning=true时rdd3的partitioner仍然为HashPartitioner；如果为false则rdd3的partitioner为None，也就是被Spark认为是随机划分的。但这个参数的限制很强，因为参与zipPartitions的rdd有多个，每个的partitioner可能不同，仅当参与的多个rdd具有相同的partitioner时preservePartitioning才有意义。
- zipWithIndeX()和zipWithUniqueId()：对rdd1中的数据进行编号，前者编号方式从0开始按序递增，生成的RDD类型是ZippedWithIndexRDD。后者编号方式为round-robin，生成的RDD类型是MapPartitionsRDD。
- subtractByKey(otherDataset)：计算出Key在rdd1中而不在rdd2中的record。此操作先将rdd1和rdd2中的<K,V> record按Key聚合在一起，得到SubtractedRDD，此过程类似cogroup。然后只保留[(a), (b)]中b为()的record，从而得到在rdd1中而不在rdd2中的元素。SubtractedRDD结构和数据依赖模式都类似于CoGroupedRDD，可以形成一对一窄依赖或Shuffle宽依赖，但实现比CoGroupedRDD更高效。
- subtract(otherDataset)：计算在rdd1中而不在rdd2中的record。与subtractByKey类似，但适用面更广，可以针对非KV类型的RDD。其底层实现基于subtractByKey来完成。先将rdd1和rdd2表示为<K,V> record，Value为null，然后按照Key将这些record聚合在一起得到SubtractedRDD，只保留[(a), (b)]中b为()的record。
- sortBy(func，[ascending], [numPartitions])：与sortByKey的语义类似，但sortBy不要求RDD类型是KV类型，只是根据每个record经过func的执行结果进行排序。sortBy基于sortByKey实现。
- glom(): 将rdd1中每个分区的record合并到一个list中。是一个简单的操作，直接将分区中的数据合并到一个list中。
### 3.4 常用的action数据操作
- action数据操作是用来对计算结果进行后处理的，同时提交计算job，经常在一连串transformation后使用。
- 判断一个操作是action还是transformation的方式是看返回值，后者一般返回RDD类型，而前者一般返回数值、数据结构（如Map）或不返回任何值（如写磁盘）。
- count(): long：统计rdd1中包含的record个数，返回long类型。
- countByKey(): Map[K,long]：统计rdd1中每个Key出现的次数，返回一个Map，要求rdd1是<K,V>类型。
- countByValue(): Map[T,long]：统计rdd中每个record出现的次数，返回一个Map。
- count操作首先计算每个分区中record的数目，然后在Driver端进行累加操作，得到最终结果。countByKey只统计每个Key出现的次数，因此首先利用mapValues操作将<K,V> record的Value设置为1（去掉原有的Value），然后利用reduceByKey统计每个Key出现的次数，最后汇总到Driver端，形成Map。countByValue操作统计每个record出现的次数，先将record变为<record, null>类型，这样接下来就可以使用reduceByKey得到每个record出现的次数，最后汇总到Driver端，形成Map。
- countByKey和countByValue需要在Driver端存放一个Map，当数据量比较大时，这个Map会超过Driver的内存大小，这时建议先使用reduceByKey对数据进行统计，然后将结果写入分布式文件系统，如HDFS等。
- collect(): Array[T]：将rdd1中的record收集到Driver端。
- collectAsMap(): Map[K,V]：将rdd1中的<K,V> record收集到Driver端，得到<K,V> Map。
- 这两个操作的逻辑比较简单，都是将RDD中的数据直接汇总到Driver端，类似count操作的流程图。在数据量较大时两者都会造成大量内存消耗。
- foreach(func): Unit和foreachPartition(func): Unit：将rdd1中的每个record按照func进行处理；将rdd1中的每个分区中的数据按照func进行处理。这俩的关系类似于map和mapPartitions的关系。但不同的是foreach操作一般会直接输出计算结果，并不形成新的RDD。
- fold(zeroValue)(func): T：将rdd1中的record按照func进行聚合，func语义与foldByKey(func)中的func相同。
- reduce(func): T：将rdd1中的record按照func进行聚合，func语义与reduceByKey(func)中的func相同。
- aggregate(zeroValue)(seqOp, combOp): U：将rdd1中的record进行聚合，seqOp和combOp的语义与aggregateByKey(zeroValue)(seqOp, combOp)中的类似。
- 上面三个操作与xxxByKey的区别在于后者会生成新的RDD，而前者直接计算出结果，并不生成新的RDD。会先在rdd1的每个分区中计算局部结果，然后在Driver端将局部结果聚合成最终结果。需要注意的是fold操作中每次聚合时初始值zeroValue都会参与计算，而foldByKey在聚合来自不同分区的record时并不使用初始值；aggregate操作中seqOp和combOp聚合时初始值zeroValue都会参与计算，而在aggregateByKey中，初始值只参与seqOp的计算。
- 虽然xxxByKey可以对每个分区中的record以及跨分区且具有相同Key的record进行聚合，但这些聚合都是在部分数据上进行的，不是针对所有record进行全局聚合，因此当我们需要全局聚合结果时需要对这些部分聚合结果进行merge，而这个merge操作就是xxxByKey对应的xxx。这几个操作的共同问题是，当需要merge的部分结果很大时，数据传输量很大，而且Driver是单点merge，存在效率和内存空间限制问题。因此，Spark对这些聚合操作进行了优化，提出了下面两个操作。
- treeAggregate(zeroValue)(seqOp, combOp, depth): U：将rdd1中的record按照树形结构进行聚合，seqOp和combOp的语义与aggregate中的相同，树的高度默认值为2。
- treeReduce(func, depth): T：将rdd1中的record按树形结构进行聚合，func的语义与reduce(func)中的相同。
- treeAggregate使用树形聚合的方法来优化全局聚合阶段，从而减轻Driver端聚合的压力（数据传输量和内存用量）。树形聚合方法类似归并排序中的层次归并。当分区数量为6时，2层树形聚合的性能足以满足要求（3个分区聚合到一个新分区）。当分区数量成百上千时，可以连续使用foldbyKey进行多层树形聚合。
- 在treeAggregate的过程中，虽然foldByKey使用宽依赖，但实际上每个分区中只存在一个record，因此形式上是宽依赖，实际上数据传输时类似多对一的窄依赖。如果输入数据中的分区个数本来就很少，比如4个，则treeAggregate也会退化为类似aggregate的方式进行处理。此时treeAggregate与aggregate的区别就是tree中zeroValue会被多次调用（由于调用了fold函数）
- treeReduce实际上是调用treeAggregate实现的，但没有初始值zeroValue，因此其逻辑处理流程图是简化版的treeAggregate。
- reduceByKeyLocality(func)：将rdd1中的record按照Key进行reduce。不同于reduceByKey，此操作首先在本地进行局部reduce，并使用HashMap来存储聚合结果，然后把数据汇总到Driver端进行全局reduce，返回的结果存放到HashMap中而不是RDD中。
- take(num): Array[T]: 将rdd1中前num个record取出，形成一个数组。
- first(): T：只取出rdd1中的第一个record，等价于take(1)。
- takeOrdered(num)：取出rdd1中最小的num个record，要求rdd1中的record是可比较的。
- top(num)：取出rdd1中最大的num个record，要求rdd1中的record是可比较的。
- take操作首先取出rdd1中第一个分区的前num个record，如果num大于分区中record的总数，则take继续从后面的分区中取出record。为了提高效率，Spark会在取第一个分区record时估计还需要对多少个后续的分区进行操作。
- takeOrdered操作首先使用map在每个分区中寻找最小的num个record，然后将这些record收集到Driver端，进行排序，再取出前num个record。top的执行逻辑与此相似，只是改为取出最大的num个record。
- 上面四种操作都需要将数据收集到Driver端，因此不适合num较大的情况。
- max(): T：计算rdd1中record的最大值。
- min(): T：计算rdd1中record的最小值。
- max和min操作都是基于reduce(func)实现，func的语义是取最大值和最小值。
- isEmpty(): Boolean：判断rdd是否为空，如果rdd不包含任何record，那么返回true。
- lookup(Key): Seq[V]：找出rdd中包含特定Key的Value，将这些Value形成list。此操作首先filter出给定Key对应的record，然后使用map得到相应的Value，最后使用collect将这些Value收集到Driver形成list。如果rdd1的partitioner已经确定，如HashPartitioner，那么在filter前就可以通过Hash(Key)确定需要操作的分区，这样可以减少操作的数据。
- saveAsTextFile(path): Unit：将rdd保存为文本文件。针对String类型，将record转化为<NullWriter, Text>类型，然后一条条输出，NullWriter的意思是空写，也就是每条输出数据只包含类型为文本的Value。
- saveAsObjectFile(path): Unit：将rdd保存为序列化对象形式的文件。针对普通对象类型，将record进行序列化，并且以每10个record为一组转化为SequenceFile<NullWritable, Array[Object]>格式，调用saveAsSequenceFile写入HDFS中。
- saveAsSequenceFile(path): Unit：将rdd保存为SequenceFile形式的文件，SequenceFile用于存放序列化后的对象。针对<K,V>类型的record，将record序列化后以SequenceFile的形式写入分布式文件系统中。
- saveAsHadoopFile(path): Unit：将rdd保存为Hadoop HDFS文件系统支持的文件。此操作中会连接HDFS，并进行必要的初始化和配置，再将文件写入HDFS中。上面三个操作都是基于此操作进行。
- 上面几个操作都是将rdd中的record进行格式转化后直接写入分布式文件系统中的，逻辑比较简单。
### 3.5 对比MapReduce，Spark的优缺点
- 从编程模型角度来说，Spark更具有通用性和易用性。
    1. 通用性：基于函数式编程思想，MapReduce将数据类型抽象为<K,V>格式，并将数据处理操作抽象为map和reduce两个算子，并为两个算子设计了固定的处理流程map-Shuffle-reduce。但这种模式只适用于表达类似foldByKey, reduceByKey, aggregateByKey的处理流程，而像cogroup，join，cartesian，coalesce的流程需要更灵活的表达方式。因此Spark转变了思路，在两方面进行了优化改进：一方面借鉴了DryadLINQ/FlumeJava的思想，将输入/输出、中间数据抽象表达为一个数据结构RDD，相当于在Java中定义的Class，然后根据不同类型中的中间数据生成不同的RDD（相当于Java中生成不同类型的Object）。这样数据结构就灵活了起来，不再拘泥于Hadoop中的<K,V>格式，并且中间数据变得可定义、可表示、可操作、可连接。另一方面通过可定义的数据依赖关系来灵活连接中间数据。在MapReduce中，数据依赖关系只有Shuffle，而Spark的数据处理操作包含多种多样的数据依赖关系，并进一步总结为了宽依赖和窄依赖（包含多种子依赖关系）。此外，Spark使用DAG图来组合数据处理操作，比map-Shuffle-reduce处理流程表达能力更强。
    2. 易用性：基于灵活的数据结构和依赖关系，Spark原生实现了很多常见的数据操作，比如MR中的map、reduceByKey，SQL中的filter、groupByKey、join、sortByKey，Pig Latin中的cogroup，集合操作union、intersection，以及特殊的zip等。由于数据结构RDD上的操作可以由Spark自动并行化，程序开发时更像在写普通程序，不用考虑本地还是分布执行。并且开发者可以更容易地将数据操作与普通程序的控制流进行结合，例如使用while语句进行RDD的迭代操作。而MapReduce中实现迭代程序比较困难，需要不断手动提交job，而Spark提供了action操作，job分割和提交都完全由Spark框架来进行，易用性进一步提高。
- 虽然Spark比MapReduce更加通用、易用，但还不能达到普通语言如Java的灵活性，具体存在两个缺点：
    1. Spark中的操作都是单向操作，单向的意思是中间数据不可修改。在普通Java程序中，数据结构中存放的数据是可以直接被修改的，而在Spark中只能生成新的数据作为修改后的结果。
    2. Spark中的操作是粗粒度的。粗粒度操作是指RDD上的操作是面向分区的，也就是每个分区上的数据操作是相同的。假设处理partition1上的数据时需要partition2的数据，并不能通过RDD的操作访问到partition2的数据，只能通过添加聚合操作来将数据汇总在一起处理，而普通Java程序的操作是细粒度的，随时可以访问数据结构中的数据。
- 上述两个缺点也是并行化设计权衡后的结果，即这两个缺点是并行化的优点，粗粒度可以方便并行执行，单向操作有利于错误容忍。

## 第4章 Spark物理执行计划
### 4.1 Spark物理执行计划概览
- MapReduce、Spark等大数据处理框架的核心思想是将大的应用拆分为小的执行任务。面对复杂的数据处理流程，Spark应该如何拆分呢？
    - 想法1：一个直观的想法是将每个具体的数据操作作为一个执行阶段stage，也就是将前后关联的RDD组成一个执行阶段。这样虽然可以解决任务划分问题，但存在多个性能问题。第一个性能问题是会产生很多个任务，导致调度的压力增加。第二个性能问题是需要存储大量的中间数据。
    - 想法2：优化想法1，通过减少任务数量。仔细观察逻辑处理流程图会发现中间数据只是暂时有用的，中间数据（RDD）产生后只用于下一步计算操作，而下一步计算操作完成后中间数据就可以被删除。那么，不如干脆将这些计算操作串联起来，只用一个执行阶段来执行这些串联的多个操作，使上一步操作在内存中生成的数据被下一步操作处理完后能够及时回收，减少内存消耗。
    - 基于上述的串联思想，接下来需要解决两个问题：
        1. 每个RDD包含多个分区，如何确定需要生成的任务数？如果RDD中的每个分区的计算逻辑相同，可以独立计算，那就可以将每个分区上的操作串联为一个task，也就是为最后的RDD的每个分区分配一个task。
        2. 如何串联操作？遇到复杂的依赖关系如宽依赖要怎么处理？比如某些操作如cogroup、join的输入数据RDD可以有多个，而输出RDD一般只有一个，这时可以将串联的顺序调整为从后向前。从最后的RDD开始向前串联，当遇到宽依赖时，将该分区所依赖的上游数据（parent RDD）及操作都纳入一个task中。然而这个方案仍然存在性能问题，当遇到宽依赖时，task包含很多数据依赖和操作，导致划分出的task可能太大，而且会出现重复计算。虽然我们可以在计算完成后缓存这些需要重复计算的数据以便后续task的计算，但这样会占用存储空间，并且使得task不能同时并行计算，降低了并行度。
    - 想法3：想法2的缺点是task会变得很大，降低并行度。问题根源是宽依赖导致的重复计算，那不如直接将宽依赖前后的计算逻辑分开，形成不同的计算阶段和任务，这样就避免了task过大的问题。Spark实际上就是基于这个思想设计的。
### 4.2 Spark物理执行计划生成方法
1. 执行步骤
    - Spark采用3个步骤生成物理执行计划：
    1. 根据action操作顺序将应用划分为作业（job）：这一步主要解决何时生成job，以及如何生成job逻辑处理流程。当应用程序出现action操作时表示应用会生成一个job，该job的逻辑处理流程为从输入数据到resultRDD的逻辑处理流程。
    2. 根据宽依赖关系将job划分为执行阶段（stage）：对于每个job，从其最后的RDD往前回溯整个逻辑处理流程，如果遇到窄依赖，则将当前RDD的parent RDD纳入，并继续往前回溯。当遇到宽依赖时停止回溯，将当前已经纳入的所有RDD按照其依赖关系建立一个执行阶段。
    3. 根据分区计算将各个stage划分为计算任务（task）：执行完第2步之后，整个job被划分了大小适中（相较于想法2中的划分方法）、逻辑分明的执行阶段stage。接下来的问题是如何生成计算任务。之前的想法是每个分区上的计算逻辑相同，而且是独立的，因此每个分区上的计算可以独立成为一个task。Spark便采用了这种策略，根据每个stage中最后一个RDD的分区个数决定生成task的个数。
2. 相关问题
    - 经过上面3个步骤，Spark可以将一个应用的逻辑处理流程划分为多个job，每个job划分为多个stage，每个stage可以生成多个task，而同一个阶段中的task可以同时分发到不同的机器并行执行。看似完美，但还有3个执行方面的问题：
    1. 如何确定一个应用内不同job、stage和task的计算顺序：job的提交时间与action被调用的时间有关，当应用程序执行到rdd.action()时就会立即将其形成的job提交给Spark。job的逻辑处理流程实际上是DAG图，划分stage后仍然是DAG。每个stage的输入数据要么是job的输入数据，要么是上游stage的输出结果，因此计算顺序从包含输入数据的stage开始顺着依赖关系依次执行，仅当上游的stage都执行完成后才执行下游的stage。stage中每个task因为是独立而且同构的，可以并行运行没有先后之分。
    2. task内部数据的存储与计算问题（流水线计算）：想法2中提出的解决方案是每计算出一个中间数据（RDD中的一个分区）就将其存放在内存中，等下一个操作处理完成并生成新的RDD中的一个分区后，回收上一个RDD在内存中的数据。虽然可以减少内存空间占用，但当某个RDD中的分区数较多时仍然会占用大量内存。进一步观察RDD分区之间的关系可以发现上游分区包含的record和下游分区包含的record之间经常存在一对一的数据依赖关系。
        - 流水线式计算的好处是可以有效地减少内存使用空间，在task计算时只需要在内存中保留当前被处理的单个record即可，不需要保存其他record或已经被处理完的record。
        - 当分区之间存在多对一关系如zipPartitions时，流水线可以直接流过zipPartitions中的iter.next()方法进行计算，zipPartitions需要在内存中保存这些中间结果直到所有record流完。有些逻辑简单的算子可以省去用集合存储中间数据比如求max值，只需要保存当前最大值即可。
        - 当分区之间存在沙漏状的一对多关系如mapPartitions时，由于下游需要等上游数据都算出后才能计算得到结果，因此上游的输出结果需要保存在内存中，当下游函数计算完中间数据的每个record后就可以对该record进行回收。
        - 当分区之间存在沙漏状的多对多关系时会退化成‘计算-回收’模式，每执行完一个操作，回收之前的中间计算结果。
        - 总结：Spark采用流水线式计算来提高task的执行效率，减少内存使用量，但对于某些需要聚合中间计算结果的操作，还是需要占用一定的内存空间，这会在一定程度上影响流水线计算的效率。
    3. task间的数据传递与计算问题：stage之间存在的依赖是宽依赖，也就是下游stage中每个task需要从parent RDD的每个分区中获取部分数据。宽依赖的数据划分方法包括Hash划分、Range划分等，要求上游stage预先将输出数据进行划分，按照分区存放，分区个数与下游task的个数一致，这个过程被称为'Shuffle Write'。下游task会将属于自己分区的数据通过网络传输获取，然后将来自上游不同分区的数据聚合在一起进行处理，这个过程被称为'Shuffle Read'。
3. stage和task命名方式
    - MapReduce中stage只包含两类：map stage和reduce stage，map stage中包含多个执行map函数的任务，被称为map task；reduce stage中包含多个执行reduce函数的任务，被称为reduce task。
    - 在Spark中，stage可以有多个，有些stage既包含类似reduce的聚合操作又包含map操作，所以不用map/reduce来命名，而是直接使用stage i来命名。只能当生成的逻辑处理流程类似MR的两个执行阶段时才会习惯性区分map/reduce stage。
    - 如果task的输出结果需要进行Shuffle Write，以便传递给下一个stage，那这些task被称为ShuffleMapTasks
    - 如果task的输出结果被汇总到Driver端或直接写入分布式文件系统，那这些task被称为ResultTasks。
4. 快速了解一个应用的物理执行计划
    - 可以利用Spark UI界面提供的信息快速分析Spark的物理执行图。包括生成的job、job中包含的stage、每个stage中Shuffle Write和Shuffle Read的数据量。还可以单击DAG Visualization查看stage之间的数据依赖关系。
    - 进入Details for stage i的界面可以看到每个stage包含的task信息，包括Shuffle Write的数据量和record条数。
### 4.3 常用数据操作生成的物理执行计划
- OneToOneDependency：
    1. map, mapValues, filter, filterByRange, flatMap, flatMapValues, sample, sampleByKey, glom, zipWithIndex, zipWithUniqueId等，针对每个record进行func操作，输出一个或多个record。
    2. mapPartitions, mapPartitionsWithIndex等，针对一个分区中的数据进行操作，输出一个或多个record。
    - 上面两类操作唯一不同是操作1每读入一条record就处理和输出一条，而操作2等到分区中的全部record都处理完后再输出record。每个task处理一个分区。
- RangeDependency：
    - 有着不同partitioner的RDD之间的union，会直接将多个RDD的分区直接合并在一起，每个task处理一个分区。
- ManyToOndeDependency：
    - shuffle为false的coalesce，partitioner相同的RDD之间的union，zip，zipPartitions等，使用多对一窄依赖将parent RDD中多个分区聚合在一起。
    - child RDD中每个分区需要从parent RDD中获取所依赖的多个分区的全部数据。
    - 此stage生成的task个数与最后RDD的分区个数相等，每个task需要同时在parent RDD中获取多个分区中的数据。
- ManyToManyDependency：
    - cartesian等，使用复杂的多对多窄依赖将parent RDD中的多个分区聚合在一起。
    - child RDD中的每个分区需要从多个parent RDD中获取所依赖分区的全部数据。此stage生成的task个数与最后的RDD分区个数相等。
- 单一ShuffleDependency：
    - partitionBy, groupByKey, reduceByKey, aggregateByKey, combineByKey, foldByKey, sortByKey, coalesce(shuffle=true), repartition, repartitionAndSortWithinPartitions, sortBy, distinct等，使用宽依赖将parent RDD中的数据进行重新划分和聚合。
    - 单一shuffle指的是child RDD只与一个parent RDD形成宽依赖。每个stage中的task个数与该stage中最后一个RDD中的分区个数相等。
    - 为了进行跨stage的数据传递，上游stage中的task将输出数据进行Shuffle Write，child stage中的task通过Shuffle Read同时获取parent RDD中多个分区中的数据。与窄依赖不同，这里从parent RDD的分区中获取的数据是划分后的部分数据。
- 多ShuffleDependency：
    - cogroup, groupWith, join, intersection, subtract, subtractByKey等，使用宽依赖将多个parent RDD中的数据进行重新划分和聚合。
    - 下游stage需要等上游stage完成后再执行，Shuffle Read获取上游stage的输出数据。
# 第三部分 典型的Spark应用
## 第5章 迭代型Spark应用
### 5.1 迭代型Spark应用的分类及特点
- 迭代型Spark应用：运行在Spark上，需要进行不断迭代才能得到最终结果的应用，比普通Spark应用更为复杂。
1. 迭代型应用有哪些：主要包括机器学习应用和图计算应用。需要在数据上不断迭代计算、不断更新中间状态，最终达到收敛的结果。例如机器学习需要在训练数据上不断迭代、不断更新模型参数，使模型的损失函数取得最小值。图计算应用需要在图数据上进行迭代计算，更新各个节点的状态，最终达到一个收敛的状态。既是数据密集型也是计算密集型。
2. 迭代型应用和非迭代型应用的编程方法的区别：前者通常与算法结合紧密，有固定的计算流程。例如机器学习应用通常使用梯度下降法求解目标函数最小值，图计算应用常常使用消息传播的方法更新节点状态。针对这些固定的计算流程可以设计出相对于MR和RDD数据操作更高层的编程模型并提供给算法开发人员编写程序。普通的非迭代型应用往往没有固定处理流程，只能依赖底层RDD、DataFrame数据操作实现
3. 迭代型应用的逻辑处理流程和物理执行计划与非迭代型应用的区别：迭代计算中的job或stage个数一般会更多，而且很多是重复出现的。另外，迭代型应用会更多地使用数据缓存来存储中间数据以供复用。
### 5.2 迭代型机器学习应用--SparkLR
- 这一部分需要先了解机器学习中的逻辑回归和梯度下降法。
- SparkLR是经典机器学习算法逻辑回归的Spark分布式版本，逻辑回归是被广泛使用的分类算法，可以使用带有分类标签的训练数据迭代训练出一个线性分类模型用于解决二分类问题。
- 逻辑回归模型的训练过程主要包含两个计算步骤：一是根据训练数据计算梯度，二是更新模型参数向量w。
- 计算梯度时需要读入每个样例{xi, yi}，代入梯度公式计算，并对计算结果进行加和。由于计算时每个样例可以独立代入公式，互不影响，此处可以使用数据并行化的方法将训练样本划分为多个部分，最后将这些梯度进行加和得到最终梯度。
- 更新模型参数向量时可以在一个节点上完成，不需要并行化。
- Spark的并行化实现流程：在初始化参数向量w后进入迭代计算阶段。SparkLR使用map和reduce两个操作来完成迭代计算。每轮迭代开始前Spark先将w广播到所有task中，每个task使用map计算自己接收到的每个{xi, yi}的梯度，并且使用reduce做本地聚合，最后Driver端再收集所有的梯度得到总梯度，并根据w的更新公式对参数向量w进行更新，用于下一轮迭代计算。
- 上述过程中提到的reduce操作与reduceByKey不同，属于action操作，并不会生成reduce stage，因此SparkLR只包含一个不断重复运行的map stage。
- SparkLR的实现方法应用于大规模数据训练时会存在不少系统性能问题：
    1. 数据聚合问题：如果task过多且每个task本地聚合后的结果（单个梯度）过大，那么统一传递到Driver端仍然会造成单点的网络瓶颈。为了解决这一问题，Spark设计了性能更好的treeAggregate操作。
    2. 参数存储问题：在大规模互联网应用中，w的维度可能是千万甚至上亿，会导致单点内存瓶颈问题，即在Driver端对w进行存储和计算时可能会出现内存溢出、计算时间过长等性能和可靠性问题。学术界和工业界提出了参数服务器的解决方案，核心思想是对参数进行划分，将其分布到多个节点上，通过一定的同步或异步更新协议（如BSP、ASP、SSP等）对参数进行更新。Spark目前还未提供参数服务器的官方实现。
### 5.3 迭代型机器学习应用--广义线性模型
- 这一部分需要先了解机器学习中的广义线性模型。
- 机器学习中的线性模型是指模型的输入和输出间存在线性关系，广义线性模型是对线性模型进行扩展，使输出的总体均值通过一个非线性函数依赖线性预测值。
- 广义线性模型统一了多种线性分类和回归模型，包括用于分类的逻辑回归、线性SVM模型，以及用于回归的线性回归、Lasso回归、Ridge回归模型等等。这些模型要解决的问题都可以被抽象为一个凸优化问题，且模型的计算过程基本相同，不同点在于这些模型具有不同的计算函数（代价函数与梯度计算公式）。
- Spark通过对这些模型的计算过程进行抽象统一，同时支持不同的计算函数，可以实现广义线性模型。在此基础上，通过实现不同的计算函数就可以构建不同的模型，避免在Spark中实现不同模型算法的重复性。
- 在计算梯度的过程中，如果正则化项或损失函数不是在每个点都对w可导，就不能采用标准的梯度下降法进行梯度下降。相比于标准的梯度下降法，子梯度下降法不能保证每一轮迭代都能使目标函数变小，所以其收敛速度相对较慢。
- SVM，中文名为支持向量机，其目标是为带标签的训练数据寻找一个分类超平面，使得超平面两端的训练数据被正确地分为两类。对于二维训练数据，分类超平面就是二维平面上的一条直线，直线两侧的训练数据被分为正负样例。对于三维训练数据，分类超平面就是三维空间中的一个平面，平面两侧的训练要数据被分为正负样例
- SVM算法的目标是最大化样本数据到分类超平面的间隔距离。因为Hinge loss损失函数可以较为精确地评价SVM模型的预测值与实际值误差，所以SVM选择使用Hinge loss损失函数。
- 分类问题与回归问题的区别是，预测的是离散的还是连续的。
- 理解了不同的线性模型后，最后的问题是如何确定模型参数向量w的迭代更新什么时候结束，即迭代计算w的收敛条件是什么。实际中常常根据w的波动程度来判断，当w不再发生明显变化时认为其已收敛。或通过设置最大的迭代轮数来减少迭代训练时间。
- 训练的具体过程与前一节中的SparkLR相似。
- 使用treeAggregate虽然可以解决Driver端单点数据聚合效率低下、内存不足的问题，但会引入更多的stage和task，而且随着stage增加，越靠下游的stage，其可并行执行的task个数越小，导致整体执行的效率下降。为了解决这个问题，Spark默认将树的层数设置为两层，避免过多执行阶段。
- 基于Spark对机器学习进行并行化实现还存在一些问题（除了前一节提到的之外）：
    1. 计算同步问题：目前Spark迭代更新w参数的方法属于同步更新，需要等待所有map/reduce任务计算完成并得到最终的梯度后再更新w。因此有可能出现慢的task影响整体进度的问题，如果某些task失败需要重启则带来的计算延时更长。如果采用异步更新，即允许运行快的task使用未更新的w进行下一轮计算，虽然可以加速，但会造成收敛速度慢的问题。为了平衡计算程度和收敛速度，一些学者提出了半异步更新协议SSP，该方法的核心思想是允许在一定时限内使用旧的参数进行计算，即参数“旧”的速度不能超过m轮。此方法可以一定程度缓解同步更新等待时长长和异步更新收敛速度慢的问题，但不能从根本上解决问题。
    2. task的频繁启停问题：每轮迭代都需要启动和停止task，如果迭代轮数太多，也会带来比较长的延迟。一个可能的方案是采用task重用技术，让task一直运行，每接收到新的数据和请求就立即开始计算。
### 5.4 迭代型图计算应用--PageRank
- 这一部分需要线性代数基础知识
- 图是计算机科学中常用的一种数据结构，能够有效地表达数据之间的复杂关联。现实世界中有很多数据都可以被抽象成图数据。例如，Web网页链接、社交关系和商品交易中的商品、买家、卖家等都可以抽象成图中的顶点或边。
- 顶点度分布算法(Degree Distribution)能够计算各个顶点的度信息，可以用于分析哪些顶点与邻居顶点的联系多，哪些顶点与邻居顶点的联系少。
- 三角形计数算法(Trianble Count)能够统计图中顶点所组成的三角形数目，可以用于检测图中的社区并衡量这些社区的凝聚力等。
- 单源点最短路径(Single Source Shortest Path)算法能够求解每个顶点到图中其他顶点的最短路径，可以用于路径规划、物流、GPS导航等。
- PageRank算法是重要的链接分析算法，可以用于评估图中哪些顶点比较重要，常见的应用是网页排序。
- 在基于PageRank的网页排序中，一个节点（网站）被链接的次数越多，说明该节点越重要。具体体现在节点的入度数量多少，以及链接来源的rank值高低。
- 从数学角度来求解每个节点的rank值：先假定每个节点有一个初始rank值，如1.0，如同每个网页当前都有100人在访问。然后，rank值开始在每个节点沿着出边自由流动，给出边链接的节点带去人数*自身rank值的rank值增量。同样的过程作用到所有节点，不断迭代下去，直到达到动态平衡，此时得到的每个节点的动态平衡值即rank值。
- 有一些节点只有入边没有出边，被称为“悬挂节点”。这些节点最终会吸收所有流量，导致动态平衡的破坏。解决这一问题的一种方法是在进入悬挂节点后有一定概率能随机跳转到其他节点，就像用户在浏览完某个页面后，通过搜索引擎随机搜索和跳转到任意其他页面一样。
- 从数学角度来看，PageRank模型可以用一个状态转移矩阵A和一个rank向量R来表示。
- A矩阵中的Aij表示从节点i跳转到节点j的概率，A矩阵有一个特殊的性质是每一列加和为1，表示从一个节点出发，跳转到本节点及其它节点的总概率为1。
- 向量Rj表示第j轮迭代后每个节点的rank值。
- 可以通过矩阵A和Rj来计算第j+1轮迭代后每个节点的rank值Rj+1 = A * Rj。
- 如果把Rj看作随迭代轮数不断变化的随机变量，那么这个rank值的计算过程可以被看作是一条马尔可夫链。由于状态转移矩阵A的特征值为1，所以根据马尔可夫链的性质，随着迭代轮数不断增加，每个节点的rank值最终会收敛，即当j足够大时，Rj+1 = ARj = Rj且收敛结果与初始的rank值无关。
- 从编程角度，PageRank的计算主要包含以下3个步骤：
    1. 初始化每个节点的rank值。
    2. 将每个节点的rank值传递给其邻居节点。
    3. 每个节点根据所有邻居发送过来的rank值计算和更新自身的rank值。
- 不断重复和迭代步骤2和3，直到每个节点的rank值达到动态平衡（不再改变或改变的值很小）。
- 基于Spark的并行化实现：
    - 主要思想是将大图切分为多个子图，然后将子图分布到不同机器上进行并行计算，在必要时进行跨机器同步计算得出结果。
    - 学术界和工业界提出了多种将大图切分为子图的图划分方法，主要包含下面两种
    1. 边划分：通过对图中某些边进行切分，得到多个图分区，每个分区包含一部分节点、节点的入边和出边（Pregel计算框架中只包含入边，GraphLab框架中包含出边和入边）、节点的邻居（Pregel框架不包含而GraphLab包含）。边划分的优点是可以保留节点的邻居信息，缺点是容易出现划分不均衡，如对于度很高的节点，其关联的边都被分到一个分区中，造成其他分区中的边可能很少。并且在某些情况中（比如在GraphLab计算框架中会保留节点的入边），边划分可能存在边冗余。
    2. 点划分：通过对图中某些点进行切分来得到多个图分区，每个分区包含一部分边，以及与边相关联的节点。PowerGraph、GraphX等框架采用点划分，被划分的节点存在多个分区中。点划分的优缺点与边划分的优缺点相反，其可将边较为平均地分配到不同机器中，但没有保留节点的邻居关系。
- Spark example包中的PageRank使用的是类似Pregel的划分方式。
- 具体实现时需要考虑下面几个关键问题：
    1. 如何对图数据进行表示、存储及访问
        - 图的表示方式有多种：邻接矩阵、邻接表、边集合等。
        - 邻接矩阵需要O(n^2)的存储空间，其中n为图中顶点个数。这种方式是当图很大时，需要消耗大量存储空间，而且容易出现稀疏问题。
        - 邻接表只存储边信息，可以降低图的存储空间，并且保存了每个顶点的邻居信息。
        - 很多图算法如PageRank都是基于邻居间的消息传播进行迭代计算，因此可以选择邻接表来存储图数据。具体方法是将边集合，即源节点和目标节点的record<sourceId, destId>集合转化成邻接表links<sourceId, list(destId)>，实现时使用groupByKey(sourceId)对<sourceId, destId>集合进行聚合即可。
        - 在初始化阶段可以对Graph edges进行map和groupByKey操作，得到邻接表，并将其缓存在内存中便于后续每轮迭代计算使用。
        - 如果图数据包含边的权重信息，则可以将<sourceId, destId>改为<sourceId, (destId, weight)>（可以参照GraphX中的图表示方法）。
        - Graph edges数据通常存放在HDFS上。Spark从HDFS上读取数据的时候可以自动进行分区，这样每个task可以处理一部分数据，不同的task可以并行运行。
    2. 如何对节点的rank值进行初始化？
        1. 获取输入图中包含的所有节点信息，可以通过对邻接表提取Key来实现。
        2. 将每个Key对应的Value设置为1.0，便得到了初始化的ranks: <sourceId, rank=1.0>。
    3. 如何进行迭代计算？
        - PageRank算法的迭代过程包含以下3个步骤（简化不考虑悬挂节点的处理问题）
        1. 分发消息，将每个节点的rank值均分到其邻居节点，可以通过将邻接表links<sourceId, list(destId)>与rank表ranks<sourceId, rank=1.0>来做join得到<sourceId, [list(destId), rank]>，再算出邻居个数n，直接输出<destId, rank/n>。直观上说是节点向每个邻居节点发送发rank/n的权重信息
        2. 收集消息，通过Spark的Shuffle阶段收集每个节点接收到的邻居消息，即通过reduceByKey(sum)操作，每个节点将收到的rank信息聚合在一起得到权重和。
        3. 对消息进行聚合计算，在上一步提到rank权重和之后做进一步处理，比如使用PageRank论文中的计算公式new rank = 0.15 + 0.85 * rank，目的是保证每个节点至少有0.15的rank。
        - 不断重复迭代上述3个步骤，直到达到最大迭代轮数或rank值收敛。
    4. PageRank形成的物理执行计划是怎样的？
        - 刚开始读取Graph Edges并使用groupByKey来生成邻接表，形成一个map stage (0)和一个reduce stage (1)。
        - 第一轮迭代时需要对邻接表links和初始化的ranks进行join操作（无需Shuffle，因为links和ranks两个RDD已经用相同的Hash划分且分区个数相同），之后用flatMap将rank值分发给邻居节点，这些操作不产生Shuffle阶段，会共用上一步的reduce stage (1)。
        - 之后会使用reduceByKey来收集rank值，产生一个新的stage (2)。
        - 在第二轮迭代时计算流程与第一轮一样，但开始和结束的边界与stage (1)并不相同。因为每次join读取的是上一轮迭代的输出结果，所以这个读取过程与上一轮迭代的输出过程共用一个stage (2)。
        - 程序最后会通过foreach来输出每个节点的rank，这也是整个程序中唯一一个action操作，因此所有stage都属于同一个job。
- 在生成邻接表后，根据邻接表将rank值分发给邻居节点，该表中的每个分区包含一部分节点以及出边到达的邻居节点，这种划分方式类似Pregel的边划分方式。

# 第四部分 大数据处理框架性能和可靠性保障机制
## 第六章 Shuffle机制
### 6.1 Shuffle的意义以及设计挑战
- Shuffle解决的问题是如何将数据重新组织，使其能够在上游和下游task之间进行传递和计算。
- 如果是单纯的数据传递，只需要将数据进行分区、通过网络传输即可。但Shuffle机制还需要进行各种类型的计算（如聚合、排序），而且数据量一般会很大，所以支持不同类型的计算和提高Shuffle的性能是一大设计难点。
1. 计算的多样性：
    - Shuffle机制分为Shuffle Write和Shuffle Read两个阶段，前者主要解决上游stage输出数据的分区问题，后者主要解决下游stage从上游stage获取数据、重新组织、并为后续操作提供数据的问题。
    - 有些操作需要聚合，如groupByKey需要在ShuffleRead中聚合
    - 有些操作需要进行combine，如reduceByKey需要在Shuffle Write端进行combine。
    - 有些操作需要排序，如sortByKey需要对Shuffle Read的数据按照Key进行排序。
    - 如何确定上述操作的执行顺序？
2. 计算的耦合性：
    - 有些操作包含用户自定义的聚合函数，如aggregateByKey(seqOp, combOp)和reduceByKey(func)的输入函数。
    - 对于Shuffle Read需要聚合的情况，具体在什么时候调用这些聚合函数呢？是先读取数据再聚合还是边读取数据边聚合？
3. 中间数据存储问题：
    - 在Shuffle机制中对数据进行分区、聚合、排序等重组操作时如何组织和存放中间数据？数据量太大内存无法存下怎么办？
- 上述问题使得Shuffle机制的设计和实现需要考虑得非常全面，但实际中很难设计出完美解决所有问题的方案。
### 6.2 Shuffle的设计思想
#### 6.2.1 解决数据分区和数据聚合问题
1. 数据分区问题：在Shuffle Write阶段，如何对map task输出结果进行分区，使得reduce task可以通过网络获取相应数据？
- 解决方案：
    1. 确定分区个数：分区个数与下游stage的task个数一致（可以由用户自定义，一般为可用CPU个数的1-2倍），默认分区个数是parent RDD的分区个数最大值。
    2. 对输出数据进行分区：对map task输出的每一个record，根据key计算partitionId。这种方法非常简单，但是不支持Shuffle Write端的combine操作。（？）
2. 数据聚合问题：在Shuffle Read阶段，如何获取上游不同task的输出数据并按照Key进行聚合？
- 解决方案：
    - 数据聚合的本质是将相同Key的record放在一起进行必要的计算，这个过程可以通过HashMap实现。
    - 先将不同task获取到的record存放到HashMap中，键是record的Key，值是相同Key的record组成的value列表。
    - 再对HashMap中的每一个键值对的value列表中的元素调用func，得到新的value列表。（针对含有func的操作，比如reduceByKey）
    - 这个方案的缺点是所有Shuffle的record都需要先存在HashMap中，占用内存空间较大。并且针对含有func的操作需要上述两个步骤完成，效率较低。
- 优化方案：
    - 对于reduceByKey等包含聚合操作func的操作，可以采取在线聚合(Online aggregation)的方式减少内存空间的占用，也就是在每个Record进入HashMap的时候就将当前的value列表取出来进行func聚合并更新相应的聚合结果写回HashMap。
    - 这个方案的优化原理是一般而言聚合函数的执行结果会小于原始数据规模，比如sum和max等。
#### 6.2.2 解决map端combine问题
- 进行combine操作的目的是减少Shuffle的数据量，只有包含聚合函数的数据操作需要进行combine，比如reduceByKey, foldByKey, aggregateByKey, combineByKey, distinct等。
- 从本质上讲combine和Shuffle Read端的聚合没有区别，只是前者聚合的是来自单一task的数据而后者是来自所有map task输出的数据。
#### 6.2.3 解决sort问题
- 排序的位置：在Shuffle Read端必须执行sort，因为从每个task获取的数据组合起来后不是按全局Key进行排序的。理论上在Shuffle Write端不需要排序，但如果进行了排序，那么Shuffle Read获取到来自不同task的数据实际上已经部分有序了，可以减少Read端排序的复杂度。
- 排序的时机：
    1. 先排序再聚合：需要先使用线性数据结构如Array，存储Shuffle Read的record，然后对Key进行排序，排序后的数据可以直接从前往后进行扫描聚合，不需要再用HashMap进行分组。这是Hadoop MapReduce采用的方案，优点是可以同时满足排序和聚合的要求，缺点是需要较大内存空间来存储线性结构，并且排序和聚合有先后顺序，无法进行在线聚合从而影响效率。
    2. 排序和聚合同时进行：使用带有排序功能的Map比如TreeMap来对中间数据进行聚合，每当Shuffle Read获取到一个record就丢进TreeMap中与现有的record进行聚合。这么做虽然可以让排序和聚合同时进行，但缺点在于TreeMap的排序操作复杂度较高，插入的时间复杂度是O(nlogn)，不适合数据规模大的情况。
    3. 先聚合再排序：维持现有的HashMap聚合方案不变，将HashMap中的record或record的引用放入线性数据结构中进行排序。优点是聚合和排序过程独立，灵活性较高，也支持在线聚合，但缺点是需要复制数据或引用，空间占用较大。
    - Spark使用的是上述第3种方案，设计了特殊的HashMap来高效完成先聚合再排序的任务。
#### 6.2.4 解决内存不足的问题
- Shuffle数据量过大导致内存放不下的时候可以使用内存+磁盘的混合存储方案，内存空间不足时将内存中的数据溢写到磁盘上（这里需要进行一次预排序），空闲出来的内存继续处理新的数据，不断重复直到数据处理完成。
- 溢写到磁盘上的数据实际上是部分聚合的结果，没有和后续的数据进行聚合，所以在进行到下一步操作之前，需要对内存和磁盘上的数据进行再次聚合，也就是全局聚合。
- 全局聚合时因为有溢写前的预排序，可以按顺序读取磁盘数据并使用归并排序来降低时间复杂度和磁盘IO。
### 6.3 Spark中Shuffle框架的设计
- **目前Spark的Shuffle机制不支持在Shuffle Write端对数据进行按Key排序**，但框架保留了支持这个操作的能力。
#### 6.3.1 Shuffle Write框架设计和实现
- Spark设计了一个通用的Shuffle Write框架：map输出 -> 数据聚合 -> 排序 -> 分区
- 大体的步骤是map task计算出record和partitionId之后将record放入类似HashMap的数据结构中进行聚合，再将HashMap中的聚合后数据放入类似Array的数据结构中进行排序，最后按照partitionId将数据写入不同的数据分区中，存放到本地磁盘上。其中聚合和排序过程是可选的。
- Spark对不同的情况进行了分类以及针对性的优化调整：
1. 不需要map端聚合和排序：
    - map依次输出record并计算其PID（分区id），Spark根据PID将record依次输出到不同的buffer中，当buffer填满时将record溢写到磁盘上的分区文件中。代码中Spark将这种方式称为BypassMergeSortShuffleWriter，即不需要进行排序的Shuffle Write方式。
    - 优点：速度快，直接将record分到对应分区中。
    - 缺点：资源消耗大，每个分区都需要一个buffer（大小由spark.Shuffle.file.buffer控制，默认为32KB），且同时需要建立多个分区文件进行溢写。除了buffer的内存消耗外，每个task打开的文件数量也会造成操作系统资源不足。因此该方案适合分区个数较少（< 200）的情况。
    - 适用的操作：groupByKey(100), partitionBy(100), sortByKey(100)等。
2. 不需要map端聚合，但需要排序：
    - 需要按照PID+Key进行排序。Spark采用的方法是建立一个Array来存放map输出的record，并对Array中的元素的Key进行精心设计，将每个<K,V> record转化为<(PID, K),V> record，再按照PID+Key对record进行排序，最后将所有record写入一个文件中，通过建立索引来标示每个分区。
    - 如果Array存放不下，会先扩容，还存放不下就将Array中的record排序后spill到磁盘上，等map输出完成后再来全局排序并写入文件中。
    - 该Shuffle模式被命名为SortShuffleWrite(KeyOrdering=true)，使用的Array被命名为PartitionedPairBuffer。
    - 优点：只需要一个Array结构就可以支持排序，且Array大小可控。同时，输出的数据已经按照PID进行排序，所以只需要一个分区文件存储，解决了BypassMergeSortShuffleWriter中建立文件数过多的问题，适用于分区数很大的情况。
    - 缺点：排序会增加计算时延。
    - 适用的操作：map端不需要聚合、Key需要排序、分区个数无限制。目前Spark没有这种操作。
3. 需要map端聚合，需要或者不需要按Key进行排序：
    - 先建立一个类似HashMap的数据结构对map输出的record进行聚合，键是PID+Key，值是经过相同combine的聚合结果。当聚合完成后再对record进行排序，如果需要按Key排序，就按PID+Key进行排序，否则只按PID进行排序，结束后写入一个分区文件中。
    - 这里的HashMap也有扩容和Spill磁盘的机制。
    - 优点：只需要一个HashMap结构即可完成聚合操作，支持不同规模的数据，适用于分区数大的情况。
    - 缺点：在内存中进行聚合的内存消耗较大，且需要额外的数组进行排序。
    - 具体实现中Spark会用一个经过特殊设计和优化的数据结构PartitionedAppendOnlyMap来代替HashMap同时进行聚合和排序的操作，相当于HashMap和Array的合体。
    - 适用的操作：reduceByKey、aggregateByKey等。
- 总结：如果应用中的数据操作不需要聚合或排序，而且分区个数少，可以直接使用BypassMergeSortShuffleWriter进行输出。为了克服这种输出方式打开文件过多、buffer分配过多的缺点，也为了支持需要按Key排序的操作，Spark提供了SortShuffleWriter，使用基于Array的方法来按PID或PID+Key进行排序，只输出单一的分区文件。最后，为了支持map端的combine操作，Spark提供了基于HashMap的SortShuffleWriter，将Array替换为类似HashMap的操作来支持聚合操作，聚合后根据PID或PID+Key进行排序并输出分区文件，这种方式被称为sort-based Shuffle Write。
#### 6.3.2 Shuffle Read框架设计和实现
- 在Shuffle Read阶段，数据操作需要3个功能：跨节点数据获取、聚合和排序。Spark设计的通用Shuffle Read框架便是基于这三个流程。
- Spark对不同的情况进行了分类以及针对性的优化调整：
1. 不需要聚合，不需要按Key进行排序：
    - 最简单的情况，只需要等若有的map task结束后，reduce task不断从各个map task获取record，并将record输出到一个buffer中（大小为spark.reducer.maxSizeInFlight=48MB）中，然后直接从buffer中获取数据即可。
    - 优点；逻辑简单、实现简单、内存消耗很小。
    - 缺点：不支持聚合、排序等复杂功能。
    - 适用的操作：partitionBy等。
2. 不需要聚合，需要按Key进行排序：
    - 需要实现数据获取和按Key排序的功能。获取数据后，将buffer中的record依次输出到一个Array结构中。由于这里采用了本来用于Shuffle Write端的PartitionedPairBuffer结构，所以还保留了每个record的PID。然后，对Array中的record按照Key进行排序，将排序结果输出或传递给下一步操作。
    - PartitionedPairBuffer会在内存不足时将record排序后spill到磁盘上，最后再进行全局排序。
    - 优点：只需要一个Array结构就可以支持按照Key进行排序，且Array大小可控，spill机制不受数据规模限制。
    - 缺点：排序增加计算时延。
    - 适用的操作：sortByKey、sortBy等
3. 需要聚合，不需要或需要按Key进行排序：
    - 获取record后，Spark建立一个类似HashMap的数据结构（ExternalAppendOnlyMap）对buffer中的record进行聚合，键是record中的key，值是相同key聚合后的结果。如果需要按照Key排序，则建立一个Array结构，读取HashMap中的record，并对它们按照Key进行排序。
    - 这里的HashMap有扩容和spill机制。
    - 优点：只需要一个HashMap和一个Array结构就可以支持reduce端的聚合和排序功能，带有spill机制的HashMap支持各种规模的数据聚合，支持在线聚合。
    - 缺点：内存中聚合带来的内存消耗、全局聚合带来的消耗。数据从HashMap拷贝到Array中进行排序的内存消耗（Spark针对这点使用了特殊数据结构ExternalAppendOnlyMap）。
    - 适用的操作：reduceByKey、aggregateByKey等。
    - 总结：总体来讲Shuffle Read与Shuffle Write框架使用的技术和数据结构类似，而且由于不需要分区，过程会比后者更加简单。
### 6.4 支持高效聚合和排序的数据结构
- PartitionedAppendOnlyMap：类似HashMap+Array，**用于map端聚合及排序**，包含PID。
- ExternalAppendOnlyMap：类似HashMap+Array，**用于reduce端聚合及排序**。
- PartitionedPairBuffer：类似Array，仅用于map和reduce端数据排序，包含PID。
- PartitionedAppendOnlyMap和ExternalAppendOnlyMap都基于AppendOnlyMap实现。
- 仔细观察Shuffle Write/Read过程，可以发现Shuffle机制中使用的数据结构的两个特征：一是只需要支持record的插入和更新操作，不需要支持删除操作；二是只有内存放不下时才需要spill到磁盘上，因此数据结构设计以内存为主、磁盘为辅。
#### 6.4.1 AppendOnlyMap的原理
- AppendOnlyMap实际上是一个只支持record添加和对Value进行更新的HashMap。与Java使用数组+链表不同，AppendOnlyMap只使用数组来存储元素，根据元素的Hash值确定存储位置，**当存储元素发生Hash值冲突时，使用二次地址探测（向后指数递增）来解决Hash冲突**。
- 扩容：当AppendOnlyMap的利用率达到70%时，扩张一倍，并对所有的Key进行rehash。
- 排序：先将数组中所有的record转移到数组的前端，再通过排序算法（如快速排序）对这些record进行排序。对于需要按Key排序的操作，如sortByKey，可以按照Key值进行排序，其他操作按照Key的Hash值进行排序即可。
- 输出：迭代数组中的record，从前到后扫描输出。
#### 6.4.2 ExternalAppendOnlyMap
- AppendOnlyMap的优点是能够将聚合和排序功能很好地结合在一起，缺点是只能使用内存，对数据规模有要求。
- ExternalAppendOnlyMap在上述基础上基于内存和磁盘实现。
- 工作原理：先持有一个AppendOnlyMap来不断接收和聚合新来的record，当快被装满时先检查一下内存剩余空间是否可以扩展，可以的话就先扩展，不然就对record进行排序并spill到磁盘上。在所有数据处理完成之前重复执行上述操作，最终形成多个spill文件。最后将AppendOnlyMap中的record和spill文件中的record进行全局聚合。
- 上述过程中有三个核心问题：
1. AppendOnlyMap的大小估计：
    - 虽然我们可以知道数组的长度和大小，但里面存放的是Key和Value的引用，并不是实际对象的大小，而且Value被不断更新会让实际大小也发生变化。一种解决方法是在每次插入record或更新record的value时扫描一下AppendOnlyMap中存放的record，计算每个record的实际对象大小并相加，但这样会很耗时，因为AppendOnlyMap往往会有多达几万甚至几百万个record。
    - Spark设计了一种增量式的高效估算算法，在每个record插入或更新时根据历史统计值和当前变化量估算最新的总大小，复杂度为O(1)。在record插入或聚合过程中会定期对当前数组中存储的所有record进行抽样，精确计算这些record的大小、个数、更新个数以及平均值等，作为历史统计值。之后，每当record插入或更新时，会根据历史统计值和历史平均的变化值增量估算总大小，详见Spark源码中的SizeTracker.estimateSize方法。臭氧会定期进行，更新统计值已获得更高的精度。
2. Spill过程与排序：
    - 当AppendOnlyMap达到内存限制时，会将record排序后写入磁盘中，以便后续通过归并排序进行全局聚合。当需要按Key排序时，如sortByKey等操作会定义Key的排序方法，此时直接按照排序方法排序即可。如果是groupByKey这种不需要定义Key的排序方法的操作，则按照Key的Hash值进行排序，这样既可以归并排序也可以不用定义Key的排序方法，但是需要解决Hash冲突的问题，也就是在冲突发生时比较Key的实际值是否相等。（冲突的情况下怎么断定排序大小？）
3. 全局聚合：
    - Spark会从所有spill文件和AppendOnlyMap中提取第1条record并组成最小堆，再不断从最小堆中提取具有相同Key的record进行聚合，直到所有的record处理完成。
- 总结：ExternalAppendOnlyMap是一个高性能的HashMap，只支持数据插入和更新，但可以同时利用内存和磁盘对大规模数据进行聚合和排序，满足了Shuffle Read阶段数据聚合、排序的需求。
#### 6.4.3 PartitionedOnlyMap
- 与ExternalAppendOnlyMap的功能和实现基本一样，唯一的区别是此处的排序Key为PID+Key，为了在Shuffle Write阶段能根据“需不需要按照Key进行排序”来进行聚合、排序、分区。
#### 6.4.4 PartitionedPairBuffer
- 本质上是基于内存和磁盘的Array，在Spill时会将数据按照PID或PID+Key进行排序，最后再进行全局排序。
### 6.5 与Hadoop MapReduce的Shuffle机制对比
- Hadoop中的Shuffle：
    - 存在于Map Stage和Reduce Stage中间。
    - map task在读取完每个record后会将它们输出到一个固定大小的spill buffer里（一般为100MB），填满后将它们按照Key排序后输出到磁盘上，类似PartitionedPairBuffer的操作，但Hadoop严格按照Key排序。
    - spill buffer中的record只能进行排序不能进行聚合。聚合阶段需要等所有的record都spill到磁盘后专门进行，用combine()将所有spill文件中的record进行全局聚合（需要进行多次，因为每次只针对某个分区的spill文件进行聚合）。
    - 在Shuffle Read时先将每个map task输出的相应分区文件通过网络获取到本地内存，如果内存放不下，就对当前内存中的record进行聚合和排序，再spill到磁盘上，最终对内存中的record和spill文件中的record进行全局聚合（与Spark类似）。
    - 优点：流程固定、阶段分明、实现相对容易；内存消耗确定（此处的内存消耗指map阶段的spill buffer和reduce阶段的merge queue，不包含用户定义的聚合函数消耗的内存）；对Key进行严格排序使得可以使用堆数据结构进行聚合，非常高效；spill机制可以适应大数据规模。
    - 缺点：强制按照Key排序对于一些不需要排序的操作如groupByKey来说增加了计算量；不支持在线聚合，需要分阶段进行，导致消耗大量的内存和磁盘空间；产生的临时文件过多，比如map stage产生的分区文件数量为map task个数乘以reduce task个数。
- 针对Hadoop的Shuffle的缺点，Spark通过对操作类型分类来应对不同操作的排序需求，避免了强制按Key排序；使用AppendOnlyMap等数据结构通过hash-based聚合实现了在线聚合；将多个分区文件合并为一个分区文件避免了临时文件数多的问题（Spark按照PID进行排序的原因）。
- 另外，由于Hadoop MapReduce采用独立阶段聚合，Spark使用在线聚合，两者的聚合函数存在一个很大的区别：Hadoop的reduce函数接收<K, list(V)>，可以对每个record中的list(V)进行任意处理，而Spark的reduce函数每接收到一个<K, V>都要进行处理，流程上会有一些受限。