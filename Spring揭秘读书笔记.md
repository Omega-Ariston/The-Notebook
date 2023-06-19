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
- 工厂方法与FactoryBean
    - 虽然对象可以通过声明接口来避免对特定接口实现类的过度耦合，但总归需要一种方式将声明依赖接口的对象与接口实现类关联起来
    - 容器可以通过依赖注入的方式解除接口与实现类之间的耦合性，但当需要依赖并实例化第三方库中的相关类时，接口与实现类的耦合性需要其他方式来避免，通常的方式是使用工厂方法模式
    - 工厂方法模式提供一个工厂类来实例化具体的接口实现类。主体对象只需要依赖工厂类，当使用的实现类变更时，只需要变更工厂类，主体对象不需要任何变动
    1. 静态工厂方法
        - 为了向使用接口的客户端对象屏蔽以后可能对接口实现类的变动，可以定义一个静态工厂方法实现类，并在静态方法（一般是getInstance）中返回接口实现类的实例对象
        - 在bean的定义中用class属性指定静态方法工厂类，factory-method属性指定工厂方法名称
        - 容器会调用该静态方法工厂类的指定静态工厂方法，也就是说注入的对象实际是接口实现类的对象实例，即方法调用后的结果，而不是静态工厂类
        - 当工厂方法需要入参时，可以通过\<constructor-arg>来传参，因为当使用静态工厂方法实现类做bean定义时，构造方法参数是针对工厂方法的，而不是静态工厂方法实现类的构造函数
    2. 非静态工厂方法
        - 与静态工厂方法不同的是，非静态工厂类需要作为正常的bean注册到容器中，在引用这个工厂对象时需要用factory-bean属性指定工厂方法所在的工厂类实例，而不是通过class属性来指定类型
        - 非静态工厂方法需要入参时，处理方式与静态工厂方法相同
    3. FactoryBean
        - 是Spring容器提供的一种可以扩展容器对象实例化逻辑的接口，本质还是一个注册到容器里的Bean
        - 当某些对象实例化过于繁琐，通过XML配置过于复杂，宁愿使用Java代码来完成实例化过程时，或者某些第三方库不能直接注册到Spring容器时，可以实现org.springframework.beans.factory.FactoryBean接口，给出自己的对象实例化逻辑代码
        - 接口只有三个方法：getObject, getObjectType, isSingleton
        - FactoryBean类型的bean定义，通过正常的id引用，容器返回的是FactoryBean所生产的对象类型，而不是FactoryBean实现本身。如果一定想要FactoryBean本身的话，在id之前加一个前缀&
- 方法注入与方法替换
    - 当实例对象A的属性持有实例对象B的引用，且对象B在A中通过getter方法返回时，即便B对应的Bean设置为prototype类型，每次返回的对象还会是同一个，因为当容器将B对象的实例注入到A中作为属性后，A会一直持有这个引用，每次调用getter方法返回的都是这个实例。如果需要每次返回不同的实例，需要下面的方式
    1. 方法注入
        - Spring容器提出的解决方式，需要让getter方法声明符合规定的格式，并在配置文件中通知容器，当该方法被调用时，每次返回指定类型的对象实例即可。
        - 方法需要能够被子类实现或者覆写（public|protected），并且无入参。因为窗口会需要进行方法注入的对象使用Cglib动态生成一个子类实现来替代当前对象
        - 通过\<lookup-method>的name属性指定需要注入的方法名，bean属性指定需要注入的对象，当方法被调用的时候，容器每次都会返回一个新的bean对象实例
    2. 其它注入方式
        -  使用BeanFactoryAware接口
            - 即使没有方法注入，只要保证每次在调用getter方法时能new一个对象实例（调用BeanFactory的getBean方法），其实也行
            - Spring框架提供的BeanFactoryAware接口，让容器在实例化实现了该接口的bean定义的过程中，将自身注入该bean，让该bean持有容器自身的BeanFactory的引用
            - 具体实施起来时，需要让被注入对象（A）声明一个BeanFactory属性，并在B属性的getter方法中调用BeanFactory的getBean方法
            - 实际上，前文所述的方法注入中动态生成的子类，完成的是类似的逻辑，只是实现细节上不同
        - 使用ObjectFactoryCreatingFactoryBean
            - 是Spring提供的一个FactoryBean实现，返回一个ObjectFactory实例，用于生产容器管理的相关对象
            - 实际上它也实现了BeanFactoryAware接口，返回的实例只是特定于与Spring容器进行交互的一个实现，好处是隔离了客户端对象对BeanFactory的直接引用
            - 具体实施起来时，需要让被注入对象（A）声明一个ObjectFactory属性和对应的setter方法，并在B属性的getter方法中调用ObjectFactory的getObject方法
    3. 方法替换
        - 方法注入通过相应方法为主体对象注入依赖对象，而方法替换更多体现在方法的实现层面上，可以灵活替换/覆盖掉原方法的实现逻辑，实现简单的方法拦截功能
        - 需要给出org.springframework.beans.factory.support.MethodReplacer的实现类，并在其中实现要替换的方法逻辑
        - 有了要替换的方法逻辑后，通过\<replaced-method>配置到目标对象的bean定义中，name属性指定原方法名，replacer属性指定新方法所在的实现类
        - 如果替换的方法存在参数，或对象存在多个重载方法时，可以在\<replaced-method>内通过\<arg-type>明确指定将要替换的方法参数类型
        - Spring AOP可以把这个功能实现得更完美
- 容器背后的秘密
    - Spring的IoC容器会加载Configuration Metadata（通常是XML格式的配置信息），然后根据这些信息绑定整个系统的对象，最终组装成一个可用的基于轻量级容器的应用系统，实现以上功能的过程基本上可以按照类似的流程划分为两个阶段：容器启动阶段和Bean实例化阶段
    - Spring的IoC容器在实现的时候，充分运用了两个实现阶段的不同特点，在每个阶段都加入了相应的容器扩展点，以便我们根据具体场景的需要加入自定义的扩展逻辑
    1. 容器启动阶段
        - 除了用代码直接加载Configuration Metadata外，大部分情况下容器都需要依赖某些工具类（如BeanDefinitionReader）对加载的Configuration进行解析和分析，并将分析后的信息编组为相应的BeanDefinition，最后把这些保存了bean定义的BeanDefinition注册到相应的BeanDefinitionRegistry
    2. Bean实例化阶段
        - 当请求方通过容器的getBean方法请求某个对象，或者因为依赖关系容器需要隐式调用getBean方法时，会触发第二阶段的活动
        - 容器会首先检查所请求的对象之前是否已经初始化，如果没有，则根据注册的BeanDefinition实例化对象，并为其注入依赖
        - 如果对象实现了某些回调接口，也会根据回调接口的要求来装配它
        - 对象装配完毕后，容器会立即将其返回请求方使用
    - 第一阶段是根据图纸装配生产线，第二阶段是用装配好的生产线来生产具体的产品
    - 第一阶段：插手容器的启动
        - Spring提供了一种叫做BeanFactoryPostProcessor的容器扩展机制，允许在容器实例化相应对象之前，对注册到容器的BeanDefinition保存的信息做相应的修改（相当于在第一阶段的最后加一道工序）
        - 可以通过实现org.springframework.beans.factory.config.BeanFactoryPostProcessor接口来实现自定义的BeanFactoryPostProcessor
        - Spring允许通过CustomEditorConfigurer来注册自定义的PropertyEditor以补助容器中默认的PropertyEditor
        - BeanFactoryPostProcessor的应用方式：
            - BeanFactory：需要在代码中手动装配BeanFactory使用的BeanFactoryPostProcessor，如果有多个BeanFactory，需要逐个添加
            - ApplicationContext：会自动识别配置文件中的BeanFactoryPostProcessor并应用它，因此仅需要在XML配置文件中将这些Processor简单配置一下即可
        - Spring提供了一些已有实现，比如PropertyPlaceholderConfigurer和PropertyOverrideConfigurer是比较常用的
        1. PropertyPlaceholderConfigurer
            - 允许在XML配置文件中使用占位符，并将这些占位符的实际值单独配置到其它的简单properties文件中来加载，形如${jdbc.url}
            - 当BeanFactory在第一阶段加载完所有配置信息时，对象的属性信息还是以占位符的形式存在，直到当PropertyPlaceholderConfigurer被应用时，它才会使用properties配置文件中的配置信息来替换相应BeanDefinition中占位符所表示的值
            - PropertyPlaceholderConfigurer也会检查Java里System类中的Properties，可以通过setSystemPropertiesMode()或者setSystemPropertiesModeName()来控制是否加载或覆盖System相应Properties的行为，默认采用SYSTEM_PROPERTIES_MODE_FALLBACK，意思是如果properties文件中找不到相应配置，会去System的Properties里找
        2. PropertyOverrideConfigurer
            - 可以通过占位符来明确表明bean定义中property与properties文件中各配置项之间的对应关系
            - 是一种隐式透明的信息替换，在原始XML配置文件中看不出哪个值被替换了，只有到PropertyOverrideConfigurer指定的配置文件中才能看出来
            - 当容器中配置的多个PropertyOverrideConfigurer对同一个bean定义的同一个property值进行处理时，最后一个将生效
            - 配置在properties文件中的信息通常以明文表示，但PropertyOverrideConfigurer的父类PropertyResourceConfigurer提供了一个protected的方法convertPropertyValue，允许子类覆盖这个方法对相应的配置项做转换，如对加密后和字符串解密后再覆盖到bean定义中
        3. CustomEditorConfigurer
            - 与前两者不同，CustomEditorConfigurer只是辅助性地将后期会用到的信息注册到容器，对BeanDefinition没有任何变动
            - 容器从XML格式文件中读取的都是字符串形式的信息，但最终应用程序会由各种类型对象构成，其中的转换过程容器需要知晓转换规则相关的信息，CustomEditorConfigurer可以传达这类信息
            - Spring内部通过JavaBean的PropertyEditor来进行String类到其它类的类型转换，需要为每种对象类型提供一个PropertyEditor
            - Spring容器内部做具体的类型转换时，会采用JavaBean框架内默认的PropertyEditor做具体的类型转换
            - Spring框架自身提供了一些PropertyEditor，大部分位于org.springframework.beans.propertyeditors包下：
                - StringArrayPropertyEditor：会将符合CSV格式的字符串转换为String数组，可以指定字符串分隔符（默认逗号），ByteArrayPropertyEditor和CharArray的也都功能类似
                - ClassEditor：根据String类型的class名称，直接将其转换成相应的Class对象，相当于Class.forName(String)的功能，可以接收String[]作为输入，实现与上一个editor一样的功能
                - FileEditor：对应java.io.File类型的PropertyEditor，同样对资源进行定位的还有InputStreamEditor和URLEditor等
                - LocaleEditor：针对java.util.Locale类型
                - PatternEditor：针对java.utl.regex.Pattern类型
            - 自定义PropertyEditor的声明
                - 可以直接继承java.beans.PropertyEditor接口，但通常会选择继承java.beans.PropertyEditorSupport类以避免实现前者的所有方法，因为往往只会用到其中的一部分方法，其它的可以不用实现
                - 如果仅仅支持单向的String到相应对象类型转换，只需要覆写方法setAsTest(String)即可，如果想支持双向转换，需要同时考虑getAsText()方法的覆写
    - 第二阶段：了解bean的一生
        - 经过第一阶段后，容器现在保存了所有对象的BeanDefinition，并时刻准备着当getBean被调用时将对应的bean实例化并返回
        - getBean方法除了被客户端对象显式调用外，在容器内还有两种隐式调用情况：
            1. 对于BeanFactory而言，对象实例化默认采用懒加载，当对象A第一次被实例化时，如果其依赖的对象B没有被实例化，则容器内部会自己调用getBean先实例化B，对客户端来说是隐式的
            2. 对于ApplicationContext而言，启动后会实例化所有bean定义，其实就是在第一阶段完成后，对注册到容器的所有bean定义调用getBean
        - 一个bean定义的getBean方法只有第一次被调用时才会去实例化这个对象，之后返回的都是容器缓存的对象
        1. Bean的实例化与BeanWrapper
            - 容器内部使用了策略模式来决定采用何种方式初始化bean实例，通常可以通过反射或CGLIB动态字节码生成
            - InstantiationStrategy定义是实例化策略的抽象接口，其直接子类SimpleInstantiationStrategy通过反射实现了简单的对象实例化功能，但**不支持方法注入**
            - CglibSubclassingInstantiationStrategy继承了SimpleInstantiationStrategy以反射方式实例化对象的功能，并且通过CGLIB的动态字节码生成功能，可以动态生成某个类的子类，因此**支持方法注入**，是默认情况下容器内部使用的策略
            - **容器不会直接返回构造完成的对象实例**，而是会用BeanWrapper对其进行包裹，返回相应的BeanWrapper实例
            - BeanWrapper的存在是为了进行对象属性的设置（避免反射），其继承了PropertyAccessor接口，可以以统一的方式对对象属性进行访问
            - BeanWrapper会使用前文提到的PropertyEditor来做类型转换和设置对象属性
        2. 各色的Aware接口
            - 上一个步骤完成后，Spring容器将当前对象实例实现的以Aware命名结尾的接口定义中规定的依赖注入给当前对象实例
            - BeanFactory类型的容器有以下几个接口
                1. BeanNameAware：会将该对象实例的bean定义对应的beanName设置到当前对象实例
                2. BeanClassLoaderAware：会将对应加载当前bean的ClassLoader注入当前对象实例，默认会使用加载org.springframework.util.ClassUtils类的ClassLoader
                3. BeanFactoryAware：BeanFactory容器会将自身设置到当前对象实例（用于前文中提到的prototype类型创建新对象实例）
            - ApplicationContext类型的容器有以下几个接口：
                1. ResourceLoaderAware：ApplicationContext实现了Spring的ResourceLoader接口，当前对象如果也实现了，会将当前ApplicationContext自身设置到当前对象实例
                2. ApplicationEventPublisherAware：会将自身注入到当前对象
                3. MessageSourceAware：通过此接口提供国际化信息支持，会将自身设置到当前对象实例
                4. ApplicationContextAware：会将自身注入当前对象实例
                - 虽然上述几个接口都是将ApplicationContext自身注入对象实例，但注入的属性是不同的
                - 这几个接口的检测与设置依赖的实现机理与BeanFactory不同，使用的是BeanPostProcessor
        3. BeanPostProcessor
            - 与容器启动阶段的BeanFactoryPostProcessor类似，BeanPostProcessor会在对象实例化阶段处理容器内所有符合条件的对象实例
            - 提供了两个主要方法：postProcessBeforeInitialization和postProcessAfterInitialization
            - 到这一步时容器会检测注册到当前对象实例的BeanPostProcessor类，然后调用其中的方法
            - 常见的场景是处理标记接口实现类，或为当前对象提供代理实现，比如在before方法中对前文的几个Aware接口做检测，并做相应的属性注入；又比如替换当前对象实例或用字节码增强当前对象实例
            - before和after针对的是下文中提到的InitializingBean与init-method步骤
            - Spring的AOP更多地使用BeanPostProessor来为对象生成相应的代理对象
            - 自定义BeanPostProcessor：
                - 通过实现BeanPostProcessor接口，并将实现类注册到容器（BeanFactory通过手工编码ConfigurableBeanFactory的addBeanPostProcessor方法，ApplicationContext通过XML配置）
                - 可以实现类似于将密文解密后设置回对象实例的功能
        4. InitializingBean和init-method
            - InitializingBean是容器内部广泛使用的一个对象生命周期标识接口
            - 在上一步的before方法结束后，会检测当前对象是否实现了InitializingBean接口，如果是，则会调用其afterPropertiesSet()方法进一步调整对象实例的状态
            - InitializingBean接口在Spring容器内部广泛使用，但业务对象实现它不太合适（太具侵入性）。因此Spring还提供了XML配置里\<bean>的init-method属性和\<beans>的default-init-method属性
        5. DisposableBean与destroy-method
            - 容器会检查**singleton类型**的bean实例，看其是否实现了DisposableBean接口，或通过\<bean>的destroy-method属性指定了对象销毁方法，如果是，就为该实例注册一个用于对象销毁的回调，在singleton类型实例销毁之前执行销毁逻辑
            - 最常见的使用场景是在Spring容器中注册数据库连接池，系统退出后，连接池应该关闭以释放相应资源
            - 需要告知容器在哪个时间点执行对象的销毁方法：
                - BeanFactory：需要在独立应用程序的主程序退出之前，或视应用场景，调用ConfigurableBeanFactory的destroySingletons()方法
                - ApplicationContext：AbstractApplicationContext提供了registerShutdownHook()方法，其底层使用Runtime类的addShutdownHook()方法调用相应bean对象的销毁逻辑，从而保证在JVM退出前销毁逻辑会被执行
            - 使用了自定义scope的对象实例销毁逻辑也应该在合适的时机被调用执行，除了prototype类型的bean实例，因为它们在实例化返回给请求方后生命周期就不归容器管理了
### 第5章 Spring IoC容器Application Context
- Spring为基本的BeanFactory类提供了XmlBeanFactory实现，也为ApplicationContext类型容器提供了以下几个实现：
    - FileSystemXmlApplicationContext：从文件系统加载bean定义以及相关资源
    - ClassPathXmlApplicationContext：从Classpath加载bean定义以及相关资源
    - XmlWebApplicationContext：用于Web应用程序
- ApplicationContext支持了BeanFactory的大部分功能，下面是一些它自有的功能：
1. 统一资源加载策略
    - Spring提出了一套基于Resource和ResourceLoader接口的资源抽象和加载策略
    1. Resource：
        - Spring框架内部使用Resource接口作为所有资源的抽象和访问接口
        - Resource接口可以根据资源的不同类型，或资源所处的不同场合，给出相应的具体实现，Spring提供了一些实现类：
            - ByteArrayResource：将字节数组提供的数据作为一种资源进行封装，如果通过InputStream的形式访问该类型的资源，该实现会根据字节数组的数据，构造相应的ByteArrayInputStream并返回
            - ClassPathResource：从Java应用程序的ClassPath中加载具体资源并进行封装，可以使用指定的类加载器或者给定的类进行资源加载
            - FileSystemResource：对java.io.File类型的封装，可以以文件或URL的形式对该类型资源进行访问
            - UrlResource：通过java.net.URL进行的具体资源查找定位的实现类，内部委派URL进行具体的资源操作
            - InputStreamResource：将给定的InputStream视为一种资源的Resource实现类，较少用
        - 也可以自己实现Resource接口，其定义了7个方法，用于查询资源状态、访问资源内容、根据当前资源创建新的相对资源。但建议继承AbstractResource类并覆盖相应方法来实现相同效果
    2. ResourceLoader：
        - 是资源查找定位策略的统一抽象，用于查找和定位资源
        - 接口方法中的getResource(String location)方法可以根据指定的资源位置定位到具体的资源实例
        - Spring给出的几个实现类：
            - DefaultResourceLoader：默认的实现类，逻辑是首先检查资源路径是否以classpath:前缀打着，如果是，则尝试构造ClassPathResource类，否则尝试通过URL来定位资源，如果没有抛出异常，则会构造UrlResource类，否则委派getResourceByPath(String)方法来定位，这个方法的默认逻辑会构造ClassPathResource类并返回
            - FileSystemResourceLoader：为了避免上述最后一条分支的不恰当处理，可以使用这个实现类，它继承自前者，但覆写了getResourceByPath方法，使之从文件系统加载资源并以FileSystemResource类型返回
        - ResourcePatternResolver——批量查找的ResourceLoader
            - 是ResourceLoader的扩展，Loader每次只能根据资源路径返回确定的单个Resource实例，而PatternResolver可以根据指定的资源路径匹配模式返回多个Resource实例
            - 在继承ResourceLoader的基础上引入了Resource[] getResources(String)方法定义，以支持路径匹配模式，并引入了一种新的协议前缀classpath*:
            - 最常用的一个实现是PathMatchingResourcePatternResolver，支持ResourceLoader级别的资源加载，支持基于Ant风格的路径匹配模式，支持PatternSolver新增的classpath*:前缀等
            - 构造上述实现类的实例时，可以指定一个ResourceLoader或使用默认的DefaultResourceLoader，内部会将匹配后确定的资源路径委派给这个ResourceLoader来查找和定位资源
    3. ApplicationContext与ResourceLoader
        - ApplicationContext继承了ResourcePatternResolver，即间接实现了ResourceLoader接口，因此它理所应当地支持了统一资源加载
        - 如果某个bean需要依赖ResourceLoader来查找定位资源，可以通过构造方法或setter为它注入一个ResourceLoader属性，但其实把ApplicationContext自己的引用传进去就能实现一样功能，前文提到的ResourceLoaderAware和ApplicationContextAware接口就可以做到这点
        - BeanFactory需要注册自定义PropertyEditor来完成String到Resource的类型转换，而ApplicationContext容器可以正确识别Resource类型并转换后注入相关对象，所以直接在\<bean>的value里写URL就行了
        - ApplicationContext启动时会注册Spring提供的ResourceEditor用于识别Resource类型
2. 国际化支持
    - 对于Java中的国际化信息处理，主要涉及两个类：java.util.Locale和java.util.ResourceBundle
    1. Locale
        - 不同的Locale代表不同的国家和地区，它们都有相应的ISO标准简写代码表示，如Locale.CHINA，代表表示为zh_CN
        - 常用的Locale都提供静态常量，不常用的则需要根据相应的国家和地区以及语言来进行构造
    2. ResourceBundle
        - 用来保存特定于某个Locale的信息
        - 通常会管理一组信息序列，它们会有一个统一的basename，比如可以用一组properties文件来分别保存不同国家地区的信息，在命名时让它们有相同的basename（前缀）即可。每个properties文件中都有相同的键来标志具体资源条目，比如menu在中文中叫菜单，在西班牙语叫menudo
        - 有了ResourceBundle对应的资源文件后，就可以通过ResourceBundle的getBundle(String baseName, Locale locale)方法取得不同Locale对应的ResourceBundle，并根据资源的键取得相应Locale的资源条目内容了
    - MessageSource与ApplicationContext
        - Spring在Java SE的国际化支持基础上进一步抽象了国际化信息的访问接口，即MessageSource
        - 通过这个接口，可以直接传入相应的Locale、资源的键以及相应参数，就可以取得相应信息，而不用先根据Locale取得Resourcebundle再从里面查询信息了
        - ApplicationContext实现了MessageSource接口，默认情况下它会委派容器中一个bean id为messageSource的MessageSource接口实现来完成其职责，如果找不到这个实现，就自己实例化一个不含任何内容的StaticMessageSource实例，以保证方法调用不出错
        - Spring提供了有三种MessageSource的实现：
            - StaticMessageSource：简单实现，可以通过编程的方式添加信息条目，多用于测试，不应该用于正式生产环境
            - ResourceBundleMessageSource：基于标准Resourcebundle实现，对AbstractMessageSource进行了扩展，提供对多个ResourceBundle的缓存以提高查询速度，对参数化信息和非参数化信息的处理进行了优化，并对用于参数化信息格式化的MessageFormat实例进行缓存，是最常用的生产环境下的实现类
            - ReloadableResourceBundleMessageSource：基于标准的ResourceBundle实现，可以通过cacheSeconds属性指定时间段，以定期刷新并检查底层的properties资源文件是否有变更，所以使用它的时候尽量避免将信息资源文件放到classpath中。其通过ResourceLoader来加载properties信息资源文件
        - ApplicationContext启动时会自动识别容器中类型为MessageSourceAware的bean定义，并将自身作为MessageSource注入相应对象实例中
        - 如果某个对象需要使用MessageSource，可以为其声明一个MessageSource依赖，然后将ApplicationContext中那个mesageSource注入给它
        - MessageSource可以独立使用，但还让ApplicationContext实现它，是因为在web应用程序中通常会公开ApplicationContext给View层，这样通过tag就可以直接访问国际化信息了
3. 容器内部事件发布
    - Spring的ApplicationContext容器提供的容器内事件发布功能是通过提供一套基于Java SE标准自定义事件类而实现的
    - 自定义事件发布
        - Java SE提供了实现自定义事件发布功能的基础类，即java.util.EventObject类和EventListener接口，自定义事件类型可以通过扩展前者实现，事件的监听器则扩展自后者
        - 事件发布者关注的主要有两点：
            1. 具体时点上自定义事件的发布：为了避免事件处理期间事件监听器的注册或移除操作影响处理过程，需要对事件发布时点的监听器列表进行一个安全复制。事件的发布是顺序执行，监听器的处理逻辑最好简短些，以避免影响性能
            2. 自定义事件监听器的管理：客户端可以根据情况决定是否需要注册或移除某个事件监听器（通过事件发布类中的注册和移除方法），如果没有提供remove事件监听器的方法，则监听器实例会一直被事件发布者引用，即使已经过期或废弃不用了，也会存在监听器列表中，造成内存泄漏
    - Spring的容器内事件发布类结构分析
        - Spring的ApplicationContext容器内部允许以ApplicationEvent的形式发布事件，容器内注册的ApplicationListener类型的bean定义会被ApplicationContext容器自动识别，它们负责监听容器内发布的所有ApplicationEvent类型事件
        - Application Event
            - Spring容器内自定义事件类型，继承自EventObject，是一个抽象类，Spring提供了三个实现：
            1. ContextClosedEvent：容器在即将关闭时发布的事件类型
            2. ContextRefreshedEvent：容器在初始化或刷新的时候发布的事件类型
            3. RequestHandledEvent：Web请求处理后发布的事件，其有一子类ServletRequestHandledEvent提供特定于Java EE的Servler相关事件
        - ApplicationListener
            - 容器内使用的自定义事件监听器接口定义，继承自EventListener
            - 容器在启动时会自动识别并加载EventListener类型bean定义，一旦容器内有事件发布，将通知这些注册到容器的EventListener
        - ApplicationContext
            - ApplicationContext继承了ApplicationEventPublisher接口，提供了publishEvent方法定义
            - ApplicationContext在实现事件发布和监听器的注册方面，使用了ApplicationEventMulticaster接口（出于灵活性和扩展性考虑）。SimpleApplicationEventMulticaster是Spring提供的一个子类实现，默认使用SyncTaskExecutor进行事件的发布（同步顺序发布），也可以提供其它类型的TaskExecutor
            - ApplicationContext在容器启动时会检查容器内是否存在bean id为applicationEventMulticaster的EventMulticaster对象实例，没有的话就默认初始化一个SimpleApplicationEventMulticaster
    - 容器内事件发布的应用
        - 主要用于单一容器内的简单消息通知和处理，并不适合分布式、多进程、多容器之间的事件通知
        - 有事件发布需求的业务类需要拥有ApplicationEventPublisher实例的注入，可以通过实现ApplicationEventPublisherAware接口或ApplicationContextAware接口
4. 多配置模块加载的简化
    - BeanFactory也有，只是做得没ApplicationContext好
    - 通常在实际开发中会将整个系统的配置信息按照某种关注点进行分割并划分到不同的配置文件中，如按照功能模块或按照系统划分层次等
    - 加载整个系统的bean定义时需要让容器同时读入所有配置文件，BeanFactory要用程序代码一个文件一个文件读，而ApplicationContext可以直接以String[]传入配置文件路径
    - ClassPathXmlApplicationContext还可以通过指定Classpath中某个类所处位置来加载相应配置文件，比如通过A.class的位置来加载同一目录下的配置文件
### 第6章 Spring IoC容器之扩展篇
1. 注解版的自动绑定（@Autowired）
    - 前面的章节中提到可以用<\bean>的autowire属性来设置自动绑定，但现在可以直接使用@Autowired注解完成一样的功能
    - @Autowire会按照类型匹配进行依赖注入，与byType类型自动绑定方式类似
    - @Autowire可以标注于类定义的多个位置：属性、构造方法和其它任意方法
    - @Autowire的原理是遍历每个bean定义，通过反射检查每个bean定义对应的类上各种可能位置上的@Autowired（属性、函数），存在的话就从当前容器管理的对象中获取符合条件的对象，设置给@Autowired标注的属性或方法
    - Spring提供的AutowiredAnnotationBeanPostProcessor会在实例化bean定义的过程中做上述事情（别忘了注册这个PostProcessor）
    - @Qualifier本质上是byName自动绑定的注解版，可以直接点名要注入的bean
    - @Qualifier可以直接标注在方法入参上，并且能够和@Autowired配合使用
2. @Autowired之外的选择——使用JSR250标注依赖注入关系
    - JSR250的@Resource和@PostConstruct以及@PreDestroy对类进行标注也可以达到依赖注入的目的
    - @Resource与@Autowired的使用方式大致相同，但前者遵循于byName自动绑定形式，而后者是byType
    - @PostConstruct和@PreDestroy不是服务于依赖注入，主要用于标注对象生命周期管理相关方法，与Spring的InitializingBean和DisposableBean接口以及配置项中的init-method和destroy-method起到类似作用
    - 使用JSR250的这些注解需要注册CommonAnnotationBeanPostProcessor到容器
    - 如果使用XSD配置文件，可以直接用\<context:annotation-config>配置将AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor、PersistenceAnnotationBeanPostProcessor和RequiredAnnotationBeanPostProcessor一并注册到容器中
3. classpath-scanning功能介绍
    - classpath-scanning功能可能从某一顶层包（base package）开始扫描，当扫描到某个类标注了相应的注解之后，会提取该类的相关信息，构建对应的BeanDefinition，并注册到容器，就不用一个个添加了
    - 这个功能只能在XSD形式的配置文件中由\<context:component-scan>启用
    - \<context:component-scan>默认扫描的注解类型是@Component。在其语义基础上细化的@Repository、@Service和@Controller也同样适用
    - 在扫描类定义并将它们添加到容器时会使用默认的命名规则（小驼峰）来生成beanName，或用@Component("name")的方式自行指定
    - \<context:component-scan>会把AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor一起注册到容器中（annotation-config属性值默认为true时会做的事情）
    - 也可以让scan功能去扫描前面提到的四个注解以外的其它注解，通过其嵌套配置项include-filter或exclude-filter
## 第三部分 Spring AOP框架
### 第7章 一起来看AOP
- AOP，全程Aspect-Oriented-Programming，中文为面向切面编程
- 使用AOP可以对类似于Logging和Security等系统需求进行模块化的组织，简化系统需求与实现之间的对比关系，使整个系统的实现更具模块化
- AOP引入了Aspect的概念，以模块化的形式对系统中的横切关注点（cross-cutting concern）进行封装。Aspect之于AOP，相当于Class之于OOP
- AOP是一种理念，需要某种语言作为概念实体，实现AOP的语言称为AOL，即Aspect-Oriented Language，其可以与系统实现语言相同，如Java，也可以不同，如扩展自Java的AOL：AspectJ
- 将AO组件集成到OOP组件的过程，在AOP中称之为织入（Weave）过程
- 静态AOP时代
    - 第一代AOP，以AspectJ为代表，特点是相应的横切关注点以Aspect形式实现后，会通过特定的编译器将实现后的Aspect编译并织入系统的静态类中。比如AspectJ会使用ajc编译器将各个Aspect以Java字节码的形式编译到系统的各个功能模块中，以达到整合Aspect和Class的目的
    - 静态AOP的优点：Aspect直接以Java字节码的形式编译到Java类中，JVM可以像通常一样加载Java类运行，不会对整个系统的运行造成性能损失
    - 静态AOP的缺点：灵活性不够，如果横切关注点需要改变织入到系统的位置，就需要重新修改Aspect定义文件，然后使用编译器重新编译并重新织入到系统中
- 动态AOP时代
    - 第二代AOP，该时代的AOP框架或产品大都通过Java语言提供的各种动态特性来实现Aspect的织入，如JBoss AOP、Spring AOP以及Nanning等AOP框架
    - AspectJ融合了AspectWerkz框架后，也引入了动态织入行为，从而成为现在Java界唯一一个同时支持静态AOP和动态AOP特性的AOP实现产品
    - 第二代AOP的AOL大都采用Java实现，各种概念实体全部都是普通的Java类，易于开发和集成
    - 动态AOP的优点：织入过程在系统运行开始之后进行，而不是预先编译到系统类中，且织入信息大都采用外部XML文件格式保存，可以在调整织入点以及织入逻辑单元的同时不改变系统其他模块，甚至在系统运行时也能动态更改织入逻辑
    - 动态AOP的缺点：因为其大都在类加载或系统运行期间采用对系统字节码进行操作的方式来完成Aspect到系统的织入，会造成一定的运行时性能损失，但随着JVM版本提升，对反射以及字节码操作技术的更好支持，性能损失在逐渐减少
- Java平台上的AOP实现机制
    1. 动态代理
        - JDK1.3后引入的动态代理机制可以在运行期间为相应的接口动态生成对应的代理对象，因此可以将横切关注点逻辑封装到动态代理的InvocationHandler中，在系统运行期间根据横切关注点需要织入的模块位置将横切逻辑织入到相应的代理类中（即以动态代理类为载体的横切逻辑）
        - 所有需要织入横切关注点逻辑的模块类都得实现相应的接口，因为动态代理机制只针对接口有效
        - Spring AOP默认情况下采用这种机制实现AOP功能。Nanning也是，只支持动态代理机制
    2. 动态字节码增强
        - JVM加载的class文件都是符合一定规则的，通常的class文件由Java源代码文件使用Javac编译器编译而成，也可以使用ASM或CGLIB等Java工具库，在程序运行期间动态构建字节码的class文件
        - 在上述前提下，可以为需要织入横切逻辑的模块类在运行期间通过动态字节码增强技术，为这些系统模块类**生成相应的子类**，将横切逻辑加到这些子类中，让应用程序在执行期间使用这些动态生成的子类，从而达到织入的目的
        - 动态字节码增强技术无需模块类实现相应接口，但如果需要扩展的类以及类中的实例方法等声明为final时，无法对其进行子类化的扩展
        - Spring AOP在无法采用动态代理机制进行AOP功能扩展时，会使用CGLIB库的动态字节码增强支持来实现AOP的功能扩展
    3. Java代码生成
        - EJB容器会根据部署描述符文件提供的织入信息，为相应的功能模块类根据描述符提供的信息生成对应的Java代码，并通过部署工具或部署接口编译Java代码生成相应的Java类
        - 比较古老的AOP实现，早期EJB容器使用得多，现在已经不用了
    4. 自定义类加载器
        - 所有Java程序的class都需要通过相应的类加载器加载到JVM
        - 可以通过自定义加载器，在加载class文件期间，将外部文件规定的织入规则和必要信息进行读取，并添加到系统模块类的现有逻辑中，将改动后的class交给JVM运行
        - 这种方式比前面几种都要强大，可以对大部分类以及相应的实例进行织入，但问题在其本身的使用。某些应用服务器会控制整个类加载体系，这种场景下可能会造成问题
        - JBoss AOP和已经并入AspectJ项目的AspectWerkz框架都是采用自定义类加载器的方式实现
    5. AOL扩展
        - 最为强大且最难掌握的一种方式
        - 需要重新学习一门扩展了旧有语言的AOL或全新的AOL
        - 具有强类型检查，可以对横切关注点要切入的系统运行时点有更全面的控制
        - 代价太大，一般懒得搞
- AOP相关术语
    1. Joinpoint
        - 系统运行前，AOP的功能模块需要织入到OOP的功能模块中，因此需要知道在系统的哪些执行点上进行织入，这些点称为Joinpoint
        - 比较常见的Joinpoint类型：
            - 方法调用
            - 方法执行
            - 构造方法调用
            - 构造方法执行
            - 字段设置
            - 字段获取
            - 异常处理执行
            - 类初始化
    2. Pointcut
        - 代表的是Joinpoint的表述方式，将横切逻辑织入当前系统的过程中，需要参照Pointcut规定的Joinpoint信息，才知道应该往系统的哪些Joinpoint上织入横切逻辑
        1. Pointcut的表述方式
            - 直接指定Joinpoint所在方法名称：功能单一，只限于支持方法级别Joinpoint的AOP框架，通常只限于Joinpoint较少且较为简单的情况
            - 正则表达式：比较普遍的Pointcut表达方式，可以归纳表述需要符合某种条件的多组Joinpoint，几乎大部分Java平台的AOP产品都支持这种表达形式
            - 使用特定的Poincut表述语言：强大而灵活，但实现起来复杂，需要设计语言语法，实现相应解释器。AspectJ使用这种方式，其提供了一种类似于正则表达式的表述语言和解释器
        2. Pointcut运算
            - Pointcut之间可以进行逻辑运算，可以从简单的Pointcut通过逻辑运行得到最终的复杂Pointcut
            - 这里的逻辑运算指的是and、or等，如Pointcut.Males || Pointcut.Females
    3. Advice
        - Advice是单一横切关注点逻辑的载体，代表将会织入Joinpoint的横切逻辑，如果Aspect相当于OOP中的Class，那Advice就相当于Class中的Method
        1. Before Advice
            - 在Joinpoint指定位置之前执行的Advice类型，通常不会中断程序执行流程，如果被织入到方法执行类型的Joinpoint则会先于方法执行
            - 通常用于做一些系统初始化的工作，如设置初始值，获取必要资源等
        2. After Advice
            - 在相应连接点之后执行，可以细分为以下三种
            1. After returning Advice：只有当前Joinpoint处执行流程正常完成后才会执行，比如方法执行正常返回而没抛异常
            2. After throwing Advice：又称Throws Advice，只有在当前Joinpoint执行过程中抛出异常的情况下才会执行，比如某个方法执行类型的Joinpoint抛出异常而没有正常返回
            3. After Advice：不管Joinpoint处执行流程是正常返回还是抛出异常都会执行，就像Java的Finally块一样
        3. Around Advice
            - 又名拦截器
            - 能对附加项目的Joinpoint进行包裹，可以在Joinpoint之前和之后都指定相应的逻辑，甚至于中断或忽略Joinpoint处原来程序流程的执行
            - 可以完成Before和After Advice的功能，但通常情况下应该让它们各司其职
            - 应用场景非常广泛，比如J2EE中Servlet规范提供的Filter功能，就是其中一种体现，可以完成类似资源初始化、安全检查之类的横切系统关注点
        4. Introduction
            - 在AspectJ中称Inter-Type Declaration，在JBoss AOP中称Mix-in
            - 与前面几种不同，不是根据横切逻辑在Joinpoint处的执行时机区分，而是根据它可以完成的功能来做区别
            - 可以为原有的对象添加新的特性或行为
            - AspectJ采用了静态织入的形式，对象使用时Introduction逻辑已经编译织入完成，理论上性能最好；JBoss AOP或Spring AOP采用动态织入AOP，Introduction性能稍差一些
    4. Aspect
        - 是对系统中横切关注点逻辑进行模块化封装的AOP概念实体，可以包含多个PointCut以及相关Advice定义
    5. 织入和织入器
        - 织入器负责完成横切关注点逻辑到系统的最终织入
        - AspectJ有专门的编译器完成织入操作，即ajc；JBoss AOP采用自定义的类加载器完成最终织入；Spring AOP使用一组类完成最终的织入操作，ProxyFactory类则是其中最通用的
    6. 目标对象
        - 符合Pointcut所指定的条件，将在织入过程中被织入横切逻辑的对象称为目标对象
### 第8章 Spring AOP概述及其实现机制
- Spring AOP属于第二代AOP，采用动态代理机制和字节码生成技术实现
- 设计模式之代理模式
    - 代理类需要持有被代理类的引用，并且实现相同的方法，代理类负责接收方法调用请求，并通过持有的引用来调用被代理类，在调用之前和之后都可以添加自行的代理逻辑
    - 静态代理：需要针对不一样的目标对象单独实现一个代理对象，当系统中存在成百上千个符合Pointcut匹配条件的目标对象时，代理对象的实现将成为灾难
    - 动态代理：JDK1.3后引入的机制，可以为指定接口在系统运行期间动态地生成代理对象
    - 动态代理机制的实现主要由Proxy类和InvocationHandler接口实现
    - InvocationHandler用于实现横切逻辑，作用与Advice一样
    - 动态代理只能对实现了相应Interface的类使用，如果某个类没有实现任何Interface，就无法使用动态代理机制为其生成相应的动态代理对象
    - 默认情况下，当Spring AOP发现目标对象实现了相应Interface，则采用动态代理机制生成代理对象实例，否则会尝试使用名为CGLIB的开源动态字节码生成类库，为目标对象生成动态的代理对象实例
    - 动态字节码生成
        - 原理是通过对目标对象进行继承扩展，为其生成相应的子类，通过覆写来扩展父类的行为，比如横切逻辑的实现
        - 使用继承的方式来扩展子类也会碰到静态代理一样的问题，因此也要借助于CGLIB这样的动态字节码生成库
        - 需要实现net.sf.cglib.proxy.Callback，不过一般会直接使用net.sf.cglib.proxy.MethodInterceptor（扩展了Callback）接口
        - **使用CGLIB的唯一限制是无法对final方法进行覆写**
        - 实现完Callback后需要用CGLIB的Enhancer为目标对象动态地生成一个子类，并将Callback中的横切逻辑附加到该子类中。Enhancer会指定需要生成的子类对应的父类，以及Callback实现
### 第9章 Spring AOP一世
1. Spring AOP中的Joinpoint
    - **Spring AOP中仅支持方法执行类型的Joinpoint**，但对于属性的装载其实可以直接通过对setter和getter方法拦截达到同样的目的
2. Spring AOP中的Pointcut
    - Spring中以接口org.springframework.app.Pointcut作为AOP框架中所有Pointcut的最顶层抽象，其定义了两个方法用于捕捉系统中的相应Joinpoint（getClassFilter和getMethodMatcher），并提供了一个TruePointcut类型实例
    - 如果Pointcut类型为TruePointcut，默认会对系统中的所有对象，以及对象上所有被支持的Joinpoint进行匹配
    - ClassFilter
        - ClassFilter接口用于对Joinpoint所处的对象进行Class级别的类型匹配，通过matches()方法。它也包含一个TrueClassFilter类型实例，表示作无差别全匹配
    - MethodMatcher
        - MethodMatcher作为Spring主要支持的方法拦截，实现比ClassFilter复杂得多
        - MethodMatcher中有两个matches()方法，其中一个会检查目标方法的入参列表，另一个不会
        - MethodMatcher中还有一个isRuntime()方法，在不需要检查入参时，会返回False，称为StaticMethodMatcher，可以在框架内部缓存同样类型的方法匹配结果，因为不用每次都检查入参
        - isRuntime()返回True时表明每次都会对入参进行匹配检查，称为DynamicMethodMatcher，所以无法对结果进行缓存，效率相对较差。每次匹配时还是会先用不检查入参的matches()方法匹配，如果匹配上了再进一步检查入参
    - 大部分情况下StaticMethodMatcher可以满足需求，最好避免使用DynamicMethodMatcher
    - 在MethodMatcher类型的基础上，Pointcut也可以分为StaticMethodMatcherPointcut和DynamicMethodMatcherPointcut两类，因为前者的性能优势，Spring提供了更多支持
    - 几种常见的Pointcut实现
        1. NameMatchMethodPointcut
            - StaticMethodMatcherPointcut的子类
            - 可以根据自身指定的一组方法名称与Joinpoint所处的方法的方法名称进行匹配
            - 无法对重载方法进行匹配，因为重载方法名字相同但入参不同，而它只会考虑方法名，不考虑入参
            - 支持使用通配符“*”进行模糊匹配，亦可使用正则
        2. JdkRegexpMethodPointcut和Perl5RegexpMethodPointcut
            - StaticMethodPointcut的子类中有一个专门提供基于正则表达式的实现分支，以抽象类AbstractRegexpMethodPointcut为统帅
            - 使用这个类时，要注意正则表达式的匹配必须匹配整个方法签名，而不仅仅是方法名（与前一个实现的区别）
            - Perl5RegexpMethodPointcut使用Jakarta ORO提供正则表达式支持，基于perl5风格的正则表达式
        3. AnnotationMatchingPointcut
            - 根据目标对象中是否存在指定类型的注解来匹配Joinpoint
            - 使用前需要声明相应的注解，包括注解的名字，以及使用的层次（类或方法）
        4. ComposablePointcut
            - Spring AOP提供的可以进行逻辑运算的Pointcut实现，可以进行Pointcut之间的“并”和“交”运算
        5. ControlFlowPointcut
            - 相较于其它Pointcut，最为特殊，在理解和使用上都麻烦些
            - 前面介绍的Pointcut指定的方法在调用时一定会织入横切逻辑，而本Pointcut可以指定当方法被具体哪个类调用时才进行拦截
    - 要实现自定义的Pointcut，通常在StaticMethodMatcherPointcut和DynamicMethodMatcherPointcut两个抽象类的基础上实现相应子类即可
    - Spring中的Pointcut实现都是普通的Java对象，因此可以通过Spring的IoC容器来注册并使用它们。不过通常在Spring AOP的过程中，不会直接将某个Pointcut注册到容器并公开给容器中的对象使用（后文会详述）
3. Spring AOP中的Advice
    - 在Spring中，Advice按照其自身实例能否在目标对象类的所有实例中共享这一标准，可以划分为两大类：per-class类型和per-instance类型
    - per-class类型的Advice
        - 该类型的Advice的实例可以在目标对象类的所有实例之间共享，通常只提供方法拦截的功能，不会为目标对象类保存任何状态或添加新的特性
        1. Before Advice
            - 最简单的Advice类型，实现的横切逻辑将在相应的Joinpoint之前执行，执行完之后程序会从Joinpoint处继续执行
            - 不会打断程序的执行流程，但必要时也可以通过抛出相应异常的形式中断程序流程
            - 使用时实现org.springframework.aop.MethodBeforeAdvice接口即可
            - 可以用于进行某些资源初始化或其它准备性工作
        2. ThrowsAdvice
            - 对应通常AOP概念中的AfterThrowingAdvice
            - 需要根据将要拦截的Throwable的不同类型在同一个ThrowsAdvice中实现多个afterThrowing方法，框架会使用Java反射机制来调用这些方法
            - 通常用于对系统中特定的异常情况进行监控，以统一的方式对所发生的异常进行处理，可以在一种名为Fault Barrier的模式中使用
        3. AfterReturningAdvice
            - 通过此Advice可以访问到当前Joinpoint的方法返回值、方法、方法参数及所在的目标对象
            - 只有方法正常返回的情况下才会执行，不适合用于处理资源清理类工作
            - 不能修改返回值，与通常的AfterReturningAdvice的特性有所出入
        4. Around Advice
            - Spring AOP没有提供After Advice以实现finally的功能，但可以通过Around Advice来实现
            - Spring中没有直接定义其对应的实现接口，而是直接采用了AOP Alliance的标准接口，即org.aopalliance.intercept.MethodInterceptor
            - 能完成前面几种Advice的功能
            - 通过invoke方法的MethodInvocation参数可以控制对相应Joinpoint的拦截行为：调用其proceed方法，可以让程序执行继续沿着调用链传播。否则程序执行会在MethodInterceptor处短路，导致Joinpoint上的调用链中断，因此要记得用proceed方法
    - per-instance类型的Advice
        - 不会在目标类所有对象实例间共享，而是会为不同的实例对象保存它们各自的状态以及相关逻辑
        - 在Spring中，Introduction是唯一一种per-instance类型的Advice
        - Introduction可以在不改动目标类定义的情况下，为目标类添加新的属性以及行为，但必须声明相应的接口以及相应的实现，再通过特定的拦截器将新的接口定义以及实现类中的逻辑附加到目标对象之上
        - Spring提供了两个现成的实现类：DelegatingIntroductionInterceptor和DelegatePerTargetObjectIntroductionInterceptor
        - 前者会使用它持有的同一个“delegate”实例供同一目标类的所有实例共享使用，而后者会在内部持有一个目标对象与相应Introduction逻辑实现类之间的映射关系
        - 与AspectJ直接通过编译器将Introduction织入目标对象不同，Spring AOP采用的是动态代理机制，因此性能要逊色不少