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
