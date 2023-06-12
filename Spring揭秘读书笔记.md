# Spring揭秘读书笔记
## 第一部分 掀起Spring的盖头来
### 第1章 Spring框架的由来
- Spring框架最初的目的主要是为了简化Java EE的企业级应用开发（主要指EJB1.x和2.x版本的重量级开发）
- Spring框架概述：
    - Core：整个Spring框架构建在Core核心模块之上，它是整个框架的基础，其中Spring提供了一个IoC容器实现，用于以依赖注入的方式管理对象之间的依赖关系。除此之外，Core模块还包括框架内部使用的各种工具类，比如基础IO工具等
    - AOP：提供了一个轻便但功能强大的AOP框架，用于增强各POJO的能力，进而补足OOP/OOSD的缺憾，其符合AOP Alliance规范，采用Proxy模式构建，与IoC容器结合可以发挥强大威力
    - 数据访问与事务管理服务：在Core和AOP的基础上提供。在数据访问支持方面Spring对JDBC API的最佳实践极大地简化了该API的使用，为各种业务流行的ORM产品提供了支持（Hibernate, iBATIS, Toplink, JPA等）。事务管理抽象层是AOP的最佳实践，提供了编程式事务管理和声明式事务管理的完备支持
    - JAVA EE服务集成服务：用于简化各种Java EE服务，如JNDI、JMS以及JavaMail等
    - Web模块：Spring框架提供了自己的一套Web MVC框架，Spring的Portlet MVC便构建于此基础之上。Spring Web MVC并不排斥其他Web框架，如Struts、WebWork以及JSF等
## 第二部分 Sring的IoC容器
### 第2章 IoC的基本概念
- IoC中文通常翻译为“控制反转”或“依赖注入”
- 引入IoC之前的程序编写逻辑：如果A对象需要依赖B对象（比如调用B对象的方法），则需要在A对象中显式新建B对象（比如在构造函数中new一个）
- IoC的理念：不必每次用到什么依赖对象都主动地去获取，因为其本质只是调用依赖对象所提供的某项服务，只要用到这个依赖对象时它能够准备就绪就足够了，比如让别人帮你准备好（IoC Service Provider）
- 引入IoC之后的程序编写逻辑：A对象与B对象之间通过IoC Service Provider来打交道，所有被注入对象（A）和依赖对象（B）由其进行管理。A需要什么直接跟IoC Service Provider打声招呼，后者会把相应的B对象注入到A对象中，为A对象提供服务。
- 从A对象的角度看，与之前直接寻求依赖对象相比，依赖对象的取得方式发生了反转，控制也从A对象转到了IoC Service Provider那里
- IoC模式中的三种依赖注入方式：
    1. 构造方法注入
        - A对象可以通过在其构造方法中声明依赖对象的**参数列表**，让外部（通常是IoC容器）知道它需要哪些依赖对象
        - IoC Service Provider会检查A对象的构造方法，取得它所需要的依赖对象列表，进而为其注入相应的对象
        - 同一个对象不可能构造两次，因此A对象的构造乃至其整个生命周期，都应由IoC Service Provider来管理
        - 优点：对象在构造完成后即进入就绪状态，可以马上使用
        - 缺点：当依赖的对象比较多时，构造方法的参数列表会很长。通过反射构造对象，如果有相同类型的参数，处理起来会比较困难，维护和使用也麻烦。而且在Java中，构造方法无法被继承或设置默认值，对于非必须的依赖对象，可能需要引入多个构造函数，维护麻烦
    2. setter方法注入
        - 对于JavaBean对象来说，通常会通过getter和setter方法来访问和修改对应的对象属性
        - A对象只要声明B对象（无需初始化）属性，并为其添加setter方法即可
        - 优点：方法可以命名，所以描述性比构造函数注入好一些，且setter方法可以被继承，允许设置默认值，有良好的IDE支持
        - 缺点：对象无法在构造完成后马上进入就绪状态
    3. 接口注入
        - 相比于前面两种方式会没那么简单明了，比较死板和繁琐，不提倡使用（实现接口是侵入性修改）
        - A对象需要实现某个接口，这个接口提供一个方法，用来为其注入B对象。IoC Service Provider最终通过这些接口来了解该为被注入对象注入什么依赖对象
- 使用IoC的好处：不会对业务对象构成很强的侵入性，对象具有更好的测试性、可重用性和扩展性
### 第3章 掌管大局的IoC Service Provider
- 虽然业务对象可以通过IoC方式声明相应的依赖，但是最终仍然需要通过某种角色或服务将这些相互依赖的对象绑定在一起，而IoC Service Provider就对应IoC场景中的这一角色
- IoC Service Provider是一个抽象出来的概念，它可以指代任何将IoC场景中的业务对象绑定到一起的实现方式，可以是一段代码或一组相关的类，甚至是比较通用的IoC框架或IoC容器实现
- Spring的IoC容器就是一个提供依赖注入服务的IoC Service Provider
- IoC Service Provider的职责：
    1. 业务对象的构建管理
        - IoC场景中的业务对象无需关心所依赖的对象如何构建如何取得，IoC Service Provider需要将对象的构建逻辑从客户端对象那里剥离出来，以免这部分逻辑污染业务对象的实现
    2. 业务对象间的依赖绑定
        - 很重要的职责，没完成的话业务对象会得不到依赖对象的响应（比如收到一个NPE）
        - IoC Service Provider通过结合之前构建和管理的所有业务对象以及各个业务对象之间可以识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候可以处于就绪状态
- IoC Service Provider管理对象间依赖关系的方式：
    1. 直接编码方式（书里有直观的代码）
        - 当前大部分IoC容器都支持直接编码方式，比如PicoContainer、Spring、Avalon等
        - 在容器启动前，通过程序编码的方式将被注入对象和依赖对象注册到容器中，并明确它们相互之间的依赖注入关系
        - 如果是接口注入，则需要在注册完相应对象后，额外地将注入接口与相应的依赖对象绑定一下才行
        - 通过程序编码让最终的IoC Service Provider得以知晓服务的“奥义”，应是管理依赖绑定关系的最基本方式
    2. 配置文件方式
        - 是一种较为普遍的依赖注入关系管理方式，最常见的是通过XML文件来管理对象注册和对象间依赖关系
        - 在程序中使用时要先读取配置文件
    3. 元数据方式
        - 代表实现是Google Guice，通过Java中的注解（Annotation）标注依赖关系
        - 注解最终也要通过代码处理来确定最终的注入关系，也可以算作是编码方式的一种特殊情况
### 第4章 Spring的IoC容器之BeanFactory
- Spring的IoC容器是一个提供IoC支持的轻量级容器，除了基本的IoC支持，还提供了相应的AOP框架支持、企业级服务集成等服务
- Spring提供了两种容器类型：
    1. BeanFactory
        - 基础类型IoC容器，提供完整的IoC服务支持
        - 默认采用懒加载策略，即只有当客户端对象需要访问容器中某个受管对象时，才对该受管对象进行初始化以及依赖注入操作
        - 容器启动初期速度相对较快，需要的资源有限，对资源有限且功能要求不严格的场景比较合适
    2. ApplicationContext
        - 在前者的基础上构建，是相对比较高级的容器实现
        - 除了前者的所有支持外还提供了其他高级特性，比如事件发布、国际化信息支持等
        - 管理的对象在该类型容器启动之后默认全部初始化并绑定完成，因此相较于BeanFactory，要求更多的系统资源，且启动时间更长一些，对系统资源充足且要求更多功能的场景比较合适
- BeanFactory，顾名思义，生产Bean的工厂，可以完成作为IoC Service Provider的所有职责，包括业务对象的注册和对象间依赖关系的绑定
- Spring框架提倡使用POJO，把每个业务对象看作一个JavaBean对象
- BeanFactory会公开一个取得组装完成的对象的方法接口（如getBean）
- BeanFactory的对象注册与依赖绑定方式：
    1. 直接编码方式
        - DefaultListableBeanFactory是BeanFactory接口的一个比较通用的实现类，除了间接地实现了BeanFactory的接口，还实现了BeanDefinitionRegistry接口，其接口定义抽象了Bean的注册逻辑，在BeanFactory实现中担当Bean注册管理的角色
        - 每一个受管的对象在容器中都会有一个BeanDefinition的实例与之对应，该实例负责保存对象的所有必要信息，包括其对应对象的class类型、是否是抽象类、构造方法参数以及其他属性等
        - RootBeanDefinition和ChildBeanDefinition是BeanDefinition的两个主要实现类
        - 实际实用时需要先声明一个BeanFactory容器实例和需要的Bean定义实例，再将Bean定义注册到容器中，并指定依赖关系（构造函数/setter）
        - 当客户端向BeanFactory请求相应对象时，会得到一个完备可用的对象实例
    2. 外部配置文件方式
        - Spring的IoC容器支持两种配置文件格式：Properties和XML，也可以自己引入其它文件格式
        - 采用此方式时，Spring的IoC容器有一个统一的处理方式，通常情况下需要根据不同的外部配置文件格式，给出相应的BeanDefinitionReader实现类，由相应实现类负责将相应配置文件内容读取并映射到BeanDefinition，再将映射后的BeanDefinition注册到一个BeanDefinitionRegistry中，之后BeanDefinitionRegistry完成Bean的注册和加载
        - 大部分工作，包括解析文件格式、装配BeanDefinition之类，都由BeanDefinitionReader的相应实现类来做，Registry只负责保管
        - 如果使用Properties配置格式，Spring提供了PropertiesBeanDefinitionReader类用于配置文件的加载，不用自己实现
        - XML配置格式是Spring支持最完整，功能最强大的表达方式。Spring提供了XmlBeanDefinitionReader类用于配置文件的加载。除此之外，Spring还在DefaultListableBeanFactory的基础上构建了简化XML格式配置加载的XmlBeanFactory实现
    3. 注解方式
        - Spring2.5之前并没有正式支持基于注解方式的依赖注入（注解功能于Java5后引入）
        - 使用@Autowired和@Component对相关类进行标记
        - @Autowired告知Spring容器需要为当前对象注入哪些依赖对象
        - @Component用于配合classpath-scanning功能（需要在Spring配置中开启）使用
        - classpath-scanning会到指定的package下面扫描标注有@Component的类，添加到容器进行管理，并根据它们所标注的@Autowired为这些类注入符合条件的依赖对象