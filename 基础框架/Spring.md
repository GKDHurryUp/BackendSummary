# Spring基础 #

## 什么是框架 ##
框：有约束，条条框框
架：能支持某些东西
具有约束性的能支撑我们完成某种功能的半成品项目

## 什么是Spring框架？ ##
Spring是一种轻量级开发框架，一般指Spring FrameWork，它是很多模块的集合：核心容器、数据访问/集成、Web、AOP、工具、消息和测试模块
1. 核心技术：控制翻转（IOC，思想）依赖注入（DI，实现方式），AOP，事件
2. 测试：模拟对象，
3. 数据访问：事务，DAO支持，JDBC，ORM
4. Web支持：SpringMVC
5. 集合：远程处理，JMS，JCA，
6. 语言：


## bean标签 ##
用于配置对象让 spring 来创建
属性：
1. id，用于获取对象。
2. class：指定类的全限定类名。用于反射创建对象。默认情况下调用无参构造函数。
3. scope：指定对象的作用范围。
	* singleton :默认值，单例的.
	* prototype :多例的.
4. init-method：指定类中的初始化方法名称。
5. destroy-method：指定类中销毁方法名称。

## bean标签的作用范围以及生命周期  ##
1. 单例对象：scope="singleton"
一个应用只有一个对象的实例。
生命周期：
	对象出生：当应用加载，创建容器时，对象就被创建了。
	对象活着：只要容器在，对象一直活着。
	对象死亡：当应用卸载，销毁容器时，对象就被销毁了。
2. 多例对象：scope="prototype"
每次访问对象时，都会重新创建对象实例。
生命周期：
	对象出生：当使用对象时，创建新的对象实例。
	对象活着：只要对象在使用中，就一直活着。
	对象死亡：当对象长时间不用时，被 java 的垃圾回收器回收了。

## 注解 ##
1. @Component
	相当于：<bean id="" class="">
	默认生成id为类首字母小写

2. @Controller、@Service、@Repository
	@AliasFor(annotation = Component.class)

3. @Autowired
	先按type注入，多个type匹配时，会把要注入的对象**变量名**作为**bean的id**，在spring容器查找，找到了可以注入，找不到就报错

4. @Qualifier
	需要@Autowire一起使用。在自动按照类型注入的基础之上，再按照 Bean 的 id 注入。但是给方法参数注入时，可以独立使用。
5. @Resource
	直接按照 Bean 的 id 注入。

6. @Value
	注入基本数据类型和 String 类型数据的

7. @PostConstruct
	用于指定初始化方法。

8. @PreDestroy
	用于指定销毁方法。

9. @Configuration
	用于指定当前类是一个 spring 配置类，当创建容器时会从该类上加载注解。获取容器时需要使用AnnotationApplicationContext
	1. @ComponentScan
		用于指定 spring 在初始化容器时要扫描的包。
	2. @Bean
		该注解只能写在方法上，表明使用此方法创建一个对象，并且放入 spring 容器
	3. @Import
		用于导入其他配置类

## bean生命周期 ##
1. 实例化一个Bean，反射newInstance；

2. 按照Spring上下文对实例化的Bean进行配置－－也就是IOC注入；

3. 如果这个Bean已经实现了BeanNameAware接口，会调用它实现的setBeanName(String)方法，此处传递的就是Spring配置文件中Bean的id值

4. 如果这个Bean已经实现了BeanFactoryAware接口，会调用它实现的setBeanFactory(setBeanFactory(BeanFactory)传递的是Spring工厂自身（可以用这个方式来获取其它Bean，只需在Spring配置文件中配置一个普通的Bean就可以）；

5. 如果这个Bean已经实现了ApplicationContextAware接口，会调用setApplicationContext(ApplicationContext)方法，传入Spring上下文（同样这个方式也可以实现步骤4的内容，但比4更好，因为ApplicationContext是BeanFactory的子接口，有更多的实现方法）；

6. 如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessBeforeInitialization(Object obj, String s)方法，BeanPostProcessor经常被用作是Bean内容的更改，并且由于这个是在Bean初始化结束时调用那个的方法，也可以被应用于内存或缓存技术；

7. 如果Bean在Spring配置文件中配置了init-method属性会自动调用其配置的初始化方法。

8. 如果这个Bean关联了BeanPostProcessor接口，将会调用postProcessAfterInitialization(Object obj, String s)方法、；

9. 当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用那个其实现的destroy()方法；

10. 最后，如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法。
![](https://images0.cnblogs.com/blog2015/685971/201507/161744300481894.jpg)

## 依赖注入方式： ##
1. **setXxx()方法**
	property标签
2. **构造函数**
	constructor-arg标签
3. **使用 p 名称空间**
	引入xmlns:p约束，使用p:属性复制

在AutowireCapableBeanFactory中有四个属性
1. int AUTOWIRE_NO = 0; 默认方式注入
2. int AUTOWIRE_BY_NAME = 1; 通过名称注入
3. int AUTOWIRE_BY_TYPE = 2; 通过类型注入
4. int AUTOWIRE_CONSTRUCTOR = 3; 通过构造方法注入，容器中能找到最多的对象进行注入


注：字面量使用value赋值，非字面量使用ref（Reference）指定bean的id赋值

## 实例化bean ##
1. 使用构造器实例化
2. 使用静态工厂方法实例化
	bean标签中factory-method是实现实例化类的静态方法
3. 使用实例化工厂方法实例化
	- 第一个bean使用的构造器方法实例化工厂类
	- 第二个bean需要制定第一个bean的id，factory-bean，factory-method

## 列举一些重要的Spring模块？ ##
- Spring Core：基础，可以说Spring其他所有的功能都依赖于该类库。主要提供IOC和DI功能。
- Spring Aspects：该模块为与AspectJ的集成提供支持。
- Spring AOP：提供面向方面的编程实现。
- Spring JDBC：Java数据库连接。
- Spring JMS：Java消息服务。
- Spring ORM：用于支持Hibernate等ORM工具。
- Spring Web：为创建Web应用程序提供支持。
- Spring Test：提供了对JUnit和TestNG测试的支持。	

## 谈谈自己对于Spring IOC和AOP的理解 ##
### IOC ###
IOC（Inversion Of Controll，控制反转）是一种设计思想，就是将原本在程序中手动创建对象的控制权，交由给Spring框架来管理。IOC在其他语言中也有应用，并非Spring特有。IOC容器是Spring用来实现IOC的载体，IOC容器实际上就是一个Map(key, value)，Map中存放的是各种对象。

将对象之间的相互依赖关系交给IOC容器来管理，并由IOC容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。IOC容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。在实际项目中一个Service类可能由几百甚至上千个类作为它的底层，假如我们需要实例化这个Service，可能要每次都搞清楚这个Service所有底层类的构造函数，这可能会把人逼疯。如果利用IOC的话，你只需要配置好，然后在需要的地方引用就行了，大大增加了项目的可维护性且降低了开发难度。

1. XML配置
	<bean>标签中指定id和class
	在<property>中指定name和value（通过setXxx()注入）
2. 注解配置

获取方式：ApplicationContext的对象中getBean()方法
1. 通过id
2. 通过类型
	如果有两个类型，报错NoUniqueBeanDefinitionException
3. 通过id+类型

创建对象是通过反射，Class.newInstance()创建的，对象需要有无参构造

BeanFactory是一个顶层接口，ApplicationContext 是它的子接口
ApplicationContext 接口的实现类
1. ClassPathXmlApplicationContext：
它是从类的根路径下加载配置文件，当前项目下的-相对路径
2. FileSystemXmlApplicationContext：
它是从磁盘路径上加载配置文件，配置文件可以在磁盘的任意位置，负责不在当前项目中的-绝对路径。
3. AnnotationConfigApplicationContext:
当我们使用注解配置容器对象时，需要使用此类来创建 spring 容器。它用来读取注解。

### AOP ###
AOP（Aspect-Oriented Programming，面向切面编程）能够将那些与业务无关，却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可扩展性和可维护性。

Spring AOP是基于动态代理的，如果要代理的对象实现了某个接口，那么Spring AOP就会使用JDK动态代理去创建代理对象；而对于没有实现接口的对象，就无法使用JDK动态代理，转而使用CGlib动态代理生成一个被代理对象的子类来作为代理。当然也可以使用AspectJ，Spring AOP中已经集成了AspectJ，AspectJ应该算得上是Java生态系统中最完整的AOP框架了

使用AOP之后我们可以把一些通用功能抽象出来，在需要用到的地方直接使用即可，这样可以大大简化代码量。我们需要增加新功能也方便，提高了系统的扩展性。日志功能、事务管理和权限管理等场景都用到了AOP。

## 你有使用过Spring吗？ ##
有使用过，主要是使用里面的IOC和AOP

我先跟您说一下这个IOC吧，IOC的话主要是用来管理一个对象的，就是将原本在程序中手动创建对象的控制权，交由给Spring框架来管理。像以前的一个很经典的MVC三层，它们各层之间的对象都存在很强的耦合，通过new来调用每一层。而使用IOC能够对这个MVC三层进行一个解耦。
具体做法有两种，一种是配置文件，另一种是基于注解
1. 在Spring的一个配置文件中取一个bean标签使用一个class属性，将这个对象加入到ioc容器中，也要取一个ID属性，方便对这个对象的一个取用，
2. 可以使用一个@Configuration这么一个类在相应的一个方法上面将return回来的对象，通过@Bean的一个注解，把它加入到一个Spring的IOC容器中，它的方法名就是一个默认的一个ID，也在注解里自己指定ID属性

在启动IOC容器的时候可以使用两个注解，
1. 一个是@Resource，取出这个对象，按名字来取的
2. 另一个是@Autowire，按类型来取的

IOC的低层的话就是使用一个Map来做这个IOC的容器

AOP的话，先说一下OOP的编程思想，OOP的话他是一个自上而下的编程思想，而AOP的话是一个横切性的编程思想。
实现的有两种方式：
1. AspectJ

2. SpringAOP的方式，它借助了AspectJ的语法风格实现了这个AOP的编程思想

1. xml配置
配置SpringAOP就要在它的Spring配置文件上面配置上它的切面，还有它需要增强的类型，以及相应的切入点
	1. 把通知类用 bean 标签配置起来，加入IOC容器
	2. 使用 aop:config 声明 aop 配置
	3. 使用 aop:aspect 配置切面
	4. 使用 aop:pointcut 配置切入点表达式
	5. aop:xxx 配置对应的通知类型（before，after）
	
2. 注解配置
当然也可以使用注解的方式进行配置，创建一个@Aspect的一个切面类，在类里面创建方法，方法里面的内容就是相应要植入到目标类的一个逻辑代码，并要加上相应增强类型的一个注解，比如说@Before，@After，然后用@PointCut来指定这个目标类上面的哪个方法执行
	
	如果使用配置类要加入@EnableAspectJAutoProxy
	
	1. 把通知类也使用注解配置（@Component）
	2. 在通知类上使用@Aspect 注解声明为切面
	3. 在增强的方法上使用注解配置通知（@Before，@After）
	4. 用@PointCut配置在目标类上面的哪个方法执行
	5. 在 spring 配置文件中开启 spring 对注解 AOP 的支持
	

使用的比较多的场景的话，就是在这个监控日志、事务控制、权限管理这方面


# Spring进阶 #
## Spring热插件项目 ##
传统AOP
安装部署：不灵活，开发实施成本比较大
功能执行：不能灵活的切换

把项目中公共的部分抽离出来，实现一个插件工程

Spring插件
1. 可插拔控制，按需进行安装卸载
2. 提供插件管理页面
3. 精准插件控制

思路：
	做一个插件的Controller，通过前端穿一个id过来，后台拿到id，根据id找到相应的jar包，再使用ClassLoader把jar加载进内存，然后根据配置的类通过反射把类创建，并增强，放入缓存以及容器中
	spring会把所有的bean遍历，排除一些情况，然后根据配置文件把对象反射实例化，然后把bean增强

流程：
1. 验证插件是否已安装
2. 去JVM缓存获取插件对应的实例
3. 若缓存中没有，查找本地文件对应jar
4. 本地没有的话，基于远程下载jar包
5. 动态将jar装载到ClassLoader（URLClassLoader---addURL保护方法，通过反射）
6. 实例化该插件对象（AOP里面一个通知对象,实现MethodBeforeAdvice接口）
7. 动态将对象织入到Spring AOP（Advised.add）

## Spring AOP 和 AspectJ ##
1. Spring AOP是一个不完整的方案，它只能应用于由 Spring 容器管理的 bean。
	 AspectJ 是原始的 AOP 技术, 目的是提供完整的 AOP 解决方案。它更健壮, 但也比 Spring AOP 复杂得多
2. Spring AOP 利用运行时织入。
	AspectJ 使用编译时和class文件加载时织入
3. Spring AOP仅支持方法执行切入点	
	AspectJ持所有切入点
4. AspectJ编译时织入，性能高很多


## Spring框架中用到了哪些设计模式 ##

1. **工厂设计模式**：Spring使用工厂模式通**过BeanFactory和ApplicationContext创建bean对象**。
2. 代理设计模式：Spring AOP功能的实现。
3. 单例设计模式：Spring中的bean默认都是单例的。
4. 模板方法模式：Spring中的jdbcTemplate、hibernateTemplate等以Template结尾的对数据库操作的类，它们就使用到了模板模式。
5. 包装器设计模式：我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。这种模式让我们可以根据客户的需求能够动态切换不同的数据源。
6. **观察者模式**：Spring**事件驱动模型**就是观察者模式很经典的一个应用。
7. **适配器模式**：**Spring AOP的增强**或通知（Advice）使用到了适配器模式、Spring MVC中也是用到了**适配器模式适配**Controller。

# 事务 #

## 编程式事务

原生JDBC API实现事务管理是所有事务管理方式的基石。Spring使用**TransactionTemplate**进行编程式事务，需要把事务管理代码嵌入到业务方法中，而且这些大部分都是重复代码，会造成大量代码冗余

  它的优点是可以定义多个数据源，来对多个数据库进行分布式事务管理

## 声明式事务

作为一种横切关注点，通过AOP方法模块化，借助SpringAOP实现声明式事务管理

### xml使用事务

1. 添加tx名字空间

   ```
   xmlns:tx="http://www.springframework.org/schema/tx"
   ```

2. 配置事务管理器：DataSourceTransactionManager关联到对应的数据源

   ```xml
   <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <!-- 让其管理到具体的数据源 -->
       <property name="dataSource" ref="dataSource"/>
   </bean>
   ```

3. 配置事务的通知引用事务管理器：tx:advice

4. 配置事务的属性：在tx:advice内部配置tx:attributes	- 在切入表达式的基础上再次进行事务设置	

5. 配置 AOP 切入点表达式

6. 配置切入点表达式和事务通知的对应关系：aop:advisor

### 注解使用

1. 配置注解类，@Bean将事务管理器：DataSourceTransactionManager关联到对应的数据源，加入到容器中

2. 主类上@EnableTransactionManagement

   或者使用半XML，开启注解扫描

   ```xml
   <tx:annotation-driven transaction-manager="transactionManager"/>
   ```

3. 使用注解@Transactional

## 传播行为

Spring事务是在数据库事务的基础上进行封装，添加了事务传播的概念

@Transactional

1. propagation：传播行为，**当一个事务方法被另一个事务方法调用时，这个事务方法应该如何进行**

   | 事务行为                     | 当前（调用者）不存在事务 | 当前（调用者）存在事务 |
   | ---------------------------- | ------------------------ | ---------------------- |
   | Propagation_Required（默认） | 新建事务                 | 使用当前事务           |
   | Propagation_Requred_New      | 新建事务                 | 当前事务挂起，新建事务 |
   | Propagation_Supports         | 非事务执行               | 使用当前事务           |
   | Propagation_Not_Supported    | 非事务执行               | 挂起当前事务           |
   | Propagation_Mandatory        | 抛出异常                 | 使用当前事务           |
   | Propagation_Never            | 非事务方式执行           | 抛出异常               |
   | Propagation_Nested           | 新建事务                 | 嵌套事务执行           |

   总结：

   1. 死活不要事务的

      Propagation_Never

      Propagation_Not_Supported

   2. 可有可无的

      Propagation_Supports

   3. 必须有事务的

      Propagation_Requred_New

      Propagation_Nested

      它被嵌套时，实际上是起的一个子事务。外面的事务rollback了，里面的也rollback。如果里面的事务用了try catch，那么出错外面不会rollback，里面rollback。如果没用，里面的抛出错，会导致里外的事务都rollback。

      Propagation_Required

      Propagation_Mandatory

      

2. Isolation

   - 默认-1

   - 读未提交 1
   - 读已提交 2
   - 可重复读 4
   - 串行化   8

3. timeout：在事务强制回滚前等待的时间

4. readOnly：指定当前事务中一系列的操作是否为只读
   若设置为只读，mysql会在请求访问数据的时候（无论读写），不加锁。

5. rollbackFor|noRollbackFor
   配置因为什么异常而回滚/不回滚

## 事务源码

1. TransactionDefinition接口中定义了5中隔离级别，7种传播行为

2. TransactionAttribute接口继承TransactionDefinition接口，额外添加rollbackOn对回滚规则拓展（处理异常）

3. PlatformTransactionManager接口，也就是我们需要配置的DataSourceTransactionManager

   ```java
   public interface PlatformTransactionManager extends TransactionManager {
       
   	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
   			throws TransactionException;
       
       void commit(TransactionStatus status) throws TransactionException;
       
   	void rollback(TransactionStatus status) throws TransactionException;
   }
   ```

4. TransactionInterceptor事务拦截器

## 事务失效问题

- Bean不是代理对象

- 入口函数不是Public

- 数据库不支持事务

- 切点配置错误

- 类自调用事务（见下）

- 异常类型操作（不支持编译器异常）

  自定义throw new Exception()

### @Transactional自调用失效问题 ###

Spring数据库事务的约定实现原理是AOP，而AOP的原理是动态代理，在自调用的过程中（this），是类自身的调用（类自身的方法），而不是代理对象去调用，那么自然就不会产生AOP

解决：

1. 一个类调用另一个类
2. 注入自己
3. 从IoC容器获得代理对象启动AOP，配置expose-proxy="true"，通过AopContext.currentProxy()获取当前类的代理对象 

```xml
<aop:aspectj-autoproxy expose-proxy="true"/>
```

# IOC底层原理 #

常说的bean工厂其实是DefaultListableBeanFactory，其顶层接口是BeanFactory

	this.beanFactory = new DefaultListableBeanFactory();
## BeanDefinition ##
BeanDefinition是一个接口，再Spring进行扫描类之后，会创建一个BeanDefinition的子类对象，如GenericBeanDefinition，然后把扫描到的类的一些信息，如name, calss, scope保存在BeanDefinition中，这其实体现了==Java中一切皆是类的思想，就和Java中使用Class类保存一些信息==

## FactoryBean和普通bean ##
1. 实例化的过程不一样
2. 所存入的容器不一样

## bean实例化 ##
bean的条件：
1. 满足bean的一个生命周期
2. 存放在singletonObejcts单例池中

1. getSingleton()， 一个参数，先从单例池中拿，拿不到并且判断是否可以到缓存池拿，拿不到返回null
2. getSingleton()，二个参数，也会先从单例池中拿，拿不到就，就会先添加到beforeSingletonCreation中，表示这个类我准备要创建了，然后调用createBean创建

## Spring如何new出来一个对象 ##
核心方法在createBeanInstance()中，先判断是否指定了factory-method方法，如果指定了会从对应的工厂内进行实例化。
有四处return的地方

1. 第一处和第二处单例模式不会运行，resolvedConstructorOrFactoryMethod只有是多例或者factory-method才会进入
2. 第三处，推断有几个构造方法（当注入模型为AUTOWIRE_NO，Spring不关心构造方法，无论有多少个构造方法都返回null），并决定使用哪一个
3. 若以上都不满足，则通过反射调用默认无参构造创建对象

## bean生命周期中的9个BeanPostProcessor ##
1. InstantiationAwareBeanPostProcessor

2. SmartInstantiationAwareBeanPostProcessor ->
    determineConstructorsFromBeanPostProcessors 判断使用哪个构造器

3. MergedBeanDefinitionPostProcessor 缓存

4. SmartInstantiationAwareBeanPostProcessor ->getEarlyBeanReference

5. InstantiationAwareBeanPostProcessor->postProcessAfterInstantiation判断是否需要填充属性

6. InstantiationAwareBeanPostProcessor->postProcessProperties处理属性

7. BeanPostProcessor->postProcessBeforeInitialization

8. BeanPostProcessor->postProcessAfterInitialization

9. DestructionAwareBeanPostProcessor 

## lookup-method ##
单例模式的bean A需要引用另外一个非单例模式的bean B，为了在我们每次引用的时候都能拿到最新的bean B，我们可以让bean A通过实现ApplicationContextWare来感知applicationContext，但是这样与Spring代码耦合了，违背了反转控制原则

Srping应用了CGLIB，Spring在初始化容器的时候对配置了lookup-method标签的bean做了特殊处理，Spring会对bean指定的class做动态代理

## 如何把自己的一个对象交给Spring管理？ ##
1. ApplicationContext提供了一个API，通过getBeanFactory().registerSingleton()
	这种方法一般使用的场景是：别的对象需要依赖这个对象，而这个对象不依赖其他的对象
2. @Bean
3. FactoryBean

# AOP底层原理 #

如果开启了Aop的话，在ApplicationContext构造函数中，会生成AspectJAutoProxyBeanDefinitionParser，将AspectJAnnotationAutoProxyCreator注册，AspectJAnnotationAutoProxyCreator继承抽象父类的postProcessAfterInitialization，会执行wrapIfNecessary方法，然后到createProxy方法，首先创建ProxyFactory，给工厂设置一些属性，我们的代理目标对象，切面表达式等等。通过工厂得到代理类，会调用createAopProxy方法，判断是否是接口，来创建JdkDynamicAopProxy，或者ObjenesisCglibAopProxy。

在调用代理对象的实际方法时，就会调用ReflectiveMethodInvocation的proceed方法，真正执行invoke

## 代理对象是什么时候生成的？ ##
getBean()先根据传入的type解析成对应的name，最终会调用一个getSingleton方法，实质上是从一个ConcurrentHashMap中返回了名字对应的一个对象

	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

因此代理对象的是在初始化Spring容器时候生成的

在AnnotationConfigApplicationContext的构造方法中会调用refresh()，refresh()是一个

最核心的方法createBean()->doCreateBean(){
	1.创建目标对象，createBeanInstance(), 生成BeanWrapper对象包含了我们的Bean对象
	2.生成代理对象，在populateBean()设置完对象属性之后，initializeBean()中applyBeanPostProcessorsAfterInitialization()中调用循环遍历所有的BeanPostProcessors
}

## 底层原理 ##
BeanFactory是一个工厂，里面还有各种属性
FactoryBean是一个小工厂，它返回的是getObject()里面返回的对象

@Autowried会根据名字找，名字找不到还会根据类型找

class ---> BeanDefinition ----> Bean

使用自己的FactoryBean生成多个BeanDefinition，

@Import将类导入到BeanDefinition，如果该类实现了ImportBeanDefinitionRegistrar接口，Spring会直接重写的方法，把我们指定的BeanDefinition注册

## Spring整合MyBatis ##

核心思路：将MyBatis代理对象 -> 注册到Bean容器中

在BeanFactoryPostProcessor中无法设置BeanDefinitionMapper，需要在它前一步，Import时进行设置，即在MapperScannerRegistrar类中

1. FactoryBean，使用MapperFactoryBean，
2. 该类实现了ImportBeanDefinitionRegistrar接口
3. 配置中使用@Import




## MyBatis为什么不使用registerSingleton？ ##
ApplicationContext提供了一个API，通过getBeanFactory().registerSingleton()，我们可以使用此方法把对象交给Spring容器。
但是使用这种方法，对象没有参与bean实例化的过程，这个对象中的各种属性是没有办法自动注入的



## 循环依赖

### 什么情况出现循环依赖问题？

1. 原型，非单例
2. 在构造方法中，进行

### 什么是循环依赖？	

如A，B两个相互引用，在构造A的生命周期时，给A填充属性时，去SingletonObejcts单例池中寻找B，但是没有找到，所以开始初始化B，而在构造B时，去SingletonObejcts单例池中寻找A，而A还未完成创建，也找不到，就会产生循环依赖

### 如何解决循环依赖？

二级缓存：

如A，B两个相互引用，在构造A的生命周期时，实例化之后得到**原始对象**，把它放到一个Map中，即缓存，给A填充属性时，去SingletonObejcts单例池中寻找B，但是没有找到，所以开始初始化B，而在构造B时，此时就可以去**原始对象Map**中取得A的原始对象，把A注入到B中，在B构造完成之后，A的属性就可以引用B对象

三级缓存：

一般来说上述没有问题，在BeanPostProcesser中，但是存在AOP，会生成动态代理对象

根据原始对象，构造一个lambda表达式放入到三级缓存singletonObjectFactories中，B寻找A时，会从三级缓存中寻找，就会执行lambda表达式，得到AOP之后的代理对象，由于这个对象还不完整，==因为A原始对象还没有进行属性填充，所以此时不能直接把A的代理对象放入singletonObjects中==，所以只能把代理对象放入二级缓存中earlySingletonObjects。

当B创建完了之后，A继续进行生命周期，而A在完成属性注入后，会按照它本身的逻辑去进行AOP，但是已经进行进行过AOP了，不会再次执行（通过earlyProxyReferences判断beanName是否在其中），随后到二级缓存中拿到AOP代理对象，并把完成了初始化的对象放入代理对象，再把AOP代理对象加入SingletonObejcts单例池



一级缓存：SingletonObejcts

二级缓存：earlySingletonObejcts

三级缓存：singletonObjectFactories

四级缓存：earlyProxyReferences，记录原始对象是否进行过AOP

## 单例bean线程不安全问题？

Spring框架里的bean获取实例的时候都是默认单例模式，而单例是类变量，线程共享，因此会出现线程不安全

首先在创建时候使用DCL，保证创建安全

在使用的时候，使用ThreadLocal为每一个线程提供一个独立的变量副本，从而隔离多个线程对数据访问的冲突

## BeanFactory和FactoryBean的区别？

- BeanFactory是bean工厂，是一个接口，它是Spring中工厂的顶层规范，定义了Ioc容器的基本形式，负责创建bean，管理bean
- FactoryBean是工厂bean，一个能生产或修饰对象生成的工厂Bean，可以返回bean的实例，我们可以通过实现该接口对bean进行额外的操作，例如根据不同的配置类型返回不同类型的bean，简化xml配置等