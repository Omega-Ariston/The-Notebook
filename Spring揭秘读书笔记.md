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
#### BeanFactory的XML配置详解
- 所有注册到容器的业务对象，在Spring中称为Bean，每个对象在XML中的映射对应一个叫\<bean>的元素，这些元素组织起来成为\<beans>
- \<beans>是XML配置文件中最顶层的元素，下面可以包含0-1个\<description>和多个\<bean>以及\<import>或者\<alias>
- \<beans>通过以下属性对\<bean>们进行管理：
    - default-lazy-init：布尔值，用于标志是否对所有\<bean>进行懒加载
    - default-autowire：可以取值no, byName, byType, constructor和autodetect，默认为no。如果使用自动绑定，用于标志全体bean使用哪种默认绑定方式
    - default-dependency-check：可以取值none, objects, simple以及all，默认为none，不做依赖检查
    - default-init-method：如果管理的\<bean>都按照某种规则，有同样名称的初始化方法，可以在这里统一指定这个初始化方法名（不用每个bean单独指定）
    - default-destroy-method：与上一条对应，表示统一指定的对象销毁方法
- \<description>
    - 用于在配置文件中写一些描述性信息，通常省略
- \<import>
    - 通常情况下，可以根据模块功能或层次关系，将配置信息分门别类地放到多个配置文件中。不同配置文件通过\<import>进行引用
    - 一般没啥用，因为容器可以同时加载多个配置，不用通过一个配置去加载别的配置
- \<alias>
    - 可以为\<bean>起别名，一般用于简化输入
- \<bean>的配置
    - id属性：用于指定beanName，是每个注册到容器的对象的唯一标识。某些场景下可以省略，比如内部\<bean>及不需要根据beanName明确依赖关系的场合
    - name属性：用于指定bean的别名，可以使用id不能使用的一些字符，如/。并且可以通过逗号、空格或冒号分隔指定多个name，和alias的作用基本相同
    - class属性：指定bean的类型，大部分情况下是必须的，仅在少数情况下不用指定，比如使用抽象配置模板时
- 依赖关系的表达
    1. 构造方法注入
        - 通过\<constructor-arg>元素指明容器为bean注入通过\<ref>引用的Bean实例
        - type属性：当被注入对象存在多个不同参数类型的构造方法时，可以用此属性指定注入的构造方法
        - index属性：当构造方法同时传入多个相同类型的参数时，可以用此属性来指定参数的顺序
    2. setter方法注入
        - \<property>元素：有一个name属性，用于指定被注入对象在实例中定义的变量名，之后在此元素中通过\<value>或\<ref>属性或内嵌的其他元素指定容器中具体的依赖对象引用或值
        - **只使用\<property>进行依赖注入时需要保证对象提供了默认的无参构造方法**
        - 构造方法注入和setter方法注入可以同时用在一个\<bean>上
    3. \<property>和\<constructor-arg>中的其它配置项（二者通用）
        1. \<value>
            - 用于为主体对象注入简单的数据类型，除了String，也可以指定Java中的原始类型和包装类型
            - 容器注入时会做适当的转换工作
            - 是最底层的元素，里面无法再嵌套使用其它元素
        2. \<ref>
            - 用于引用容器中其他的对象实例
            - 通过ref的local、parent和bean属性来指定引用对象的beanName
            - 上述三个属性的区别在于：local只能指定与当前配置的对象在同一个配置文件的对象定义名称（可以获得XML解析器的id约束验证支持）；parent只能指定位于当前容器的父容器中定义的对象引用；bean则基本上通吃，所以一般用这个就可以了
            - \<ref>的定义为\<!ELEMENT ref EMPTY>，因此下面没有其他子元素可用了
        3. \<idref>
            - 在指定依赖对象时，如果使用名称而不是引用，可以使用\<value>进行指定，但这种场合下使用\<idref>最合适，因为容器在解析配置时会帮你检查beanName是否存在，而不用等运行时发现不存在（比如输错名字导致）
        4. 内部\<bean>
            - 使用内嵌的\<bean>可以将对象定义私有化（局限于当前对象），其它对象无法通过\<ref>引用到它
            - 配置上与普通\<bean>别无二致
        5. \<list>
            - 对应注入对象类型为java.util.List及其子类，或数组类型的依赖对象
            - 可以有序地为当前对象注入以collection形式声明的依赖
        6. \<set>
            - 对应注入对象类型为java.util.Set或者其子类的依赖对象
            - 和List区别在于Set中的元素是无序的
        7. \<map>
            - 对应注入对象类型为java.util.Map及其子类的依赖对象
            - 可以通过指定的key来获取相应的值
            - 可以内嵌多个\<entry>，每个都需要指定一个key和一个value
            - key通过\<key>或\<key-ref>来指定，前者用于通常的简单类型key，后者用于指定对象的引用作key
            - value可以通过前面提到的那些元素或\<props>来指定。简单的原始类型或单一的对象引用可以直接用\<value>或\<value-ref>
        8. \<props>
            - 简化后的\<map>，对应注入对象类型为java.util.Properties的依赖对象
            - 只能指定String类型的key和value
            - 可以嵌套多个\<prop>，每个\<prop>通过key属性指定键，内部直接写值，不用属性指定
            - 内部没有任何元素可以使用，只能指定字符串
        9. \<null/>
            - 空元素，使用场景不多，比如需要给String类型注入null值时可以用（什么都不写会返回""）
    4. depends-on
        - 通常情况可以使用之前提到的所有元素来显式地指定bean之间的依赖关系，容器在初始化当前bean定义时会根据这些元素所标记的依赖关系，先去实例化当前bean定义依赖的其它bean定义
        - depends-on用于非显式指定对象间的依赖关系（不通过类似\<ref>的元素明确指定）
        - 当对象有多个非显式依赖关系时，可以在\<bean>的depends-on属性中通过逗号分隔各个beanName
    5. autowire
        - autowire属性可以根据bean定义的某些特点将相互依赖的某些bean直接自动绑定，无需手工明确指定该bean定义相关的依赖关系，包括5种模式：
        1. no：容器默认的自动绑定模式，不采用任何形式的自动绑定，完全依赖手工明确配置依赖关系
        2. byName：按照类中声明的实例变量的名称，与XML配置文件中声明的bean定义的beanName的值进行匹配
        3. byType：容器会根据当前bean定义类型，分析其相应的依赖对象类型，在容器管理的所有bean定义中寻找与依赖对象类型相同的bean定义。匹配到多个类型相同的bean时会报错
        4. constructor：byName和byType是针对property的自动绑定，而constructor类型是针对构造方法参数的类型进行绑定，它同样是byType类型绑定模式，所以匹配到多个类型相同的bean时也会报错
        5. autodetect：byType和constructor模式的结合体，如果对象有默认的无参构造方法，容器会优先考虑byType模式，否则会使用constructor模式。如果通过构造方法注入绑定后还有剩余没绑定的属性，容器会用byType进行绑定
        - 自动绑定的缺点：依赖关系不够清晰；可能导致系统行为异常或不可预知（比如byType模式下新增了一个相同类型的bean，byName模式下修改属性名）；可能无法获得某些工具的良好支持，比如Spring IDE
    6. dependency-check
        - 此功能与自动绑定功能结合使用，用于保证自动绑定完成后，确认每个对象所依赖的对象是否按照预期一样被注入，手动绑定的场合也能用
        - 可以指定容器检查某种类型的依赖，基本上有如下4种类型：
        1. none：默认值，不做依赖检查
        2. simple：容器会对简单属性类型及相关的collection进行依赖检查，不检查对象引用类型
        3. object：只检查对象引用类型
        4. all：将simple与object结合，两个都检查
        - 总体而言，控制得力的话，依赖检查的功能基本可以不考虑使用
    7. lazy-init
        - 主要用于针对ApplicationContext容器的bean初始化行为施以更多控制
        - ApplicationContext容器会在启动时对所有的“Singleton的bean定义”进行实例化操作，如果想改变某个bean的默认实例化时机，可以通过lazy-init属性进行控制
        - 定义了该属性的bean并不一定会延迟初始化，比如当它被其它非延迟初始化的bean依赖时
        - 可以在\<beans>上进行统一设置
- \<bean>的继承
    - 前文所述的依赖均为bean之间的横向依赖，实际上bean也存在面向对象思想中的继承关系，即纵向依赖
    - 声明子类bean时可以使用parent属性来指定其父类bean，这样就继承了父bean中定义的默认值，只需要将特定的属性进行更改，而不用自己全部重新定义一遍
    - 父bean中可以使用abstract属性来将bean定义模板化，此时这个bean不会被实例化（也不必指定class属性，想指定也行）。延伸一点，如果不想让容器实例化某个对象，也可以把它设置为abstract
- bean的scope
    - BeanFactory的职责之一是负责对象的生命周期管理
    - scope用来声明容器中的对象所应该处的限定场景或该对象的存活时间，容器在对象进入其scope前生成并装配这些对象，在对象不再处于scope的限定后通常会销毁这些对象
    - Spring容器的scope只有在支持Web应用的ApplicationContext中使用才是合理的，scope有三种：
    1. singleton
        - 表示在Spring的IoC容器中只会存在一个实例，所有对该对象的引用共享一个实例
        - 该实例从容器启动，并因为第一次被请求而初始化后，会一直存活到容器退出，与IoC容器几乎拥有相同寿命
        - 和Java中Singleton设计模式的区别在于后者只保证在同一个ClassLoader中只存在一个同类型实例
    2. prototype
        - 表示容器在接到该类型对象请求时，每次都会重新生成一个新的对象实例给请求方
        - 对象实例返回给请求方后，容器不再拥有当前返回对象的引用，请求方自行负责该对象的后继生命周期管理，包括对象的销毁
    3. request、session和global session
        - 只适用于Web应用程序，通常与XmlWebApplicationContext共同使用
        - request：XmlWebApplicationContext会为每个HTTP请求创建一个全新的RequestProcessor对象供当前请求使用，请求结束后该对象实例的生命周期结束，可以看作是prototype的一种特例，只是场景更加具体
        - session：XmlWebApplicationContext会为每个独立的session创建属于它们自己的UserPreferences对象实例，对于Web应用而言，放到session中最普遍的就是用户的登录信息
        - global session：也是创建UserPreferences实例对象，但只有应用在基于portlet的Web应用程序中才有意义，映射到portlet的global范围的session。普通的基于servlet的Web应用中使用此scope时，容器会作为普通session scope对待
    4. 自定义scope类型
        - 默认的singleton和prototype是硬编码到代码中，后续的其它scope类型都属于可扩展的scope行列，需要实现org.springframework.beans.factory.config.Scope接口，实现自己的scope类型也需要实现这个接口
        - 有了Scope的实现类后，需要把Scope注册到容器中才能使用，通常情况下使用ConfigurableBeanFactory的registerScope方法注册。
        - 除了编码注册之外，Spring提供了一个专门用于统一注册自定义scope的BeanFactoryPostPrcessor实现，即org.springframework.beans.factory.config.CustomScopeConfigurer。对于ApplicationContext而言，因为它会自动识别并加载BeanFactoryPostProcessor，所以可以直接在配置文件中通过CustomScopeConfigurer来注册自定义scope（通过bean定义的class属性）