# MyBatis

是一个orm（Object Relational Mapping）框架，把数据库表字段映射为对象的属性

## JDBC的连接步骤

1. 引入数据库驱动的依赖

2. 数据库驱动的类加载到虚拟机中

   ``` java
   Class.forName("com.mysql.jdbc.Driver");
   ```

3. 得到数据库连接

   ```java
   connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/recordsystem?useSSL=false", "root", "");
   ```

4. 创建Statement

   ```java
   statement = connection.createStatement();
   ```

5. 执行sql，得到结果集

   ```java
   resultSet = statement.executeQuery("select * from user");
   ```

6. 映射到实体类中

7. 关闭资源

   要反序关闭ResultSet，Statement，Connection，因为它们之间是包装关系，一个包着另一个，如果先关闭Statement只是 ResultSet 对象无效，ResultSet 所占用的资源可能还没有释放

## 对MyBatis的理解

mybatis 是一个优秀的基于 java 的持久层框架，它内部封装了 jdbc，使开发者只需要关注 sql 语句本身， 而不需要花费精力去处理加载驱动、创建连接、创建 statement、释放资源等繁杂的过程。

mybatis 通过 xml 或注解的方式将要执行的各种 statement 配置起来，并通过代理拦截到对应的sql语句，将 java 对象和 statement 中 sql 的**动态参数进行映射生成**最终执行的 sql 语句，最后由 mybatis 框架执行 sql 并将结果映射为 java 对象并返回

## PrepareStatement中execute、executeQuery、executeUpdate

- execute执行返回的是boolean值
  - 若为true，代表查询语句，返回结果是ResultSet
  - 若为false，代表更新语句，返回结果是行数
- executeQuery，执行查询，返回的是ResultSet结果集

- executeUpdate，执行更新，返回DML操作影响数据表的行数

## Mybatis的自增主键怎么设置，如何设置表的关联？

自增主键：

在mapper.xml中，需要在表配置文件中的插入sql中配置自增长主键(useGeneratedKeys=”true” keyProperty=”id”)

表的关联：

resultType 属性可以指定结果集的类型，它支持基本类型和实体类类型。实体类中的属性名称必须和查询语句中的列名保持一致，否则无法实现封装

resultMap 标签可以建立查询的列名和实体类的属性名称不一致时建立对应关系，从而实现封装。

## MyBatis中#{}和${}的区别

\#{}是预编译处理，MyBatis会将SQL中的#{}替换成"?"，然后调用PreparedStatement的set方法来赋值，传入字符串后，会在值两边加上单引号，可以防止SQL注入

$ { }是字符串替换， MyBatis在处理$ { }时，它会将sql中的${ }替换为变量的值，传入的数据不会加两边加上单引号

## MyBatis有几种分页方式 ##
逻辑分页：使用自带的RowBounds分页，一次性将所有数据查询出来，然后通过计算offset和limit，在内存中进行筛选
物理分页：

1. 自己手写SQL分页

2. 使用分页插件PageHelper，去数据库查询指定条数的分页数据形式

   通过PageInterceptor实现mybatis一个Interceptor拦截器，拦截Executor的query方法，当我们需要对某个查询进行分页查询时，我们可以在调用Mapper进行查询前调用一次`PageHelper.startPage(..)`，这样`PageHelper`会把分页信息存入一个`ThreadLocal`变量中。

   在拦截到`Executor`的`query`方法执行时会从对应的`ThreadLocal`中获取分页信息，获取到了，则进行分页处理，处理完了后又会把`ThreadLocal`中的分页信息清理掉，以便不影响下一次的查询操作。

## MyBatis有哪些执行器？

### SimpleExecutor

每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象。

### ReuseExecutor

执行update或select，以sql作为key查找Statement对象，存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map内，供下一次使用。

### BatchExecutor

执行update（没有select，JDBC批处理不支持select），将所有sql都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后，等待逐一执行executeBatch()批处理。

## Mybatis中如何指定使用哪一种Executor执行器？

1. 在Mybatis配置文件中，可以指定默认的ExecutorType执行器类型

2. 手动给DefaultSqlSessionFactory的创建SqlSession的方法传递ExecutorType类型参数

## Mapper接口的工作原理是什么？

Mapper接口没有具体的实现类，当调用接口方法时，接口的全限定名+方法名拼接字符串作为key，可唯一定位一个MappedStatement，在MyBatis中，每一个`<select>、...<>`标签都会解析为一个MappedStatement对象。

接口里的方法是不能重载的，因为是使用全限名+方法名保存和寻找策略

Mapper接口的工作原理是JDK动态代理，MyBatis运行时会使用JDK动态代理为Dao接口生成代理proxy对象，代理对象会拦截接口的方法，转而执行MappedStatement所代表的的sql，然后将sql执行结果返回

## MyBatis二级缓存

一级缓存：同一个SqlSession对象（PerpetualCache）

MyBatis默认开启一级缓存，如果同样的SqlSession对象查询相同的数据，则只会在第一次查询时向数据库发送Sql语句，并将查询的结果放入到SqlSession的Map中，后续再次查询直接从缓存中查找即可

在Executor中执行query方法，到具体实现类BaseExecutor中执行，queryFromDatabase方法，查询完成后执行localCache.putObject(key, list)就是放入一个Map中，key是CacheKey包含了hashcode，以及配置的id，sql语句，value就是查询出来对象的List集合。

BaseExecutor 在每次执行 update 方法的时候，都会先 clearLocalCache() 清除缓存，

二级缓存：同一个namespace生成的mapper对象（TransactionalCache）

MyBatis默认关闭二级缓存，namespace就是配置的接口的全类名（包名.类名），通过Mapper接口生成动态代理对象，namespace就决定了Mapper对象的产生，那么它们就会共享二级缓存，当配置了二级缓存，会使用 CachingExecutor 装饰 Executor，通过TransactionalCacheManager管理二级缓存，进入一级缓存的查询流程前，先在CachingExecutor 进行二级缓存的查询。当session关闭时，通过序列号存入硬盘中

开启二级缓存：

1. xml配置文件

   ```xml
   <setting name="cacheEnabled" value="true"/>
   ```

2. mapper.xml中

   ```xml
   <cache/>
   ```

   如何sql写在注解上，使用@CacheNamespaceRef

localcache

# MyBatisPlus

MyBatisPlus简称MP，是一个MyBatis的增强工具，在MyBatis的基础上只做增强不做改变，为简化开发、提升效率而生

## 一、主键策略

MyBatisPlus默认使用雪花算法生成主键

在属性上，可以添加@TableId(type = IdType.AUTO)，来指明主键的生成策略

```java
public enum IdType {
    /**
     * 数据库ID自增
     */
    AUTO(0),
    /**
     * 该类型为未设置主键类型
     */
    NONE(1),
    /**
     * 用户输入ID
     * 该类型可以通过自己注册自动填充插件进行填充
     */
    INPUT(2),

    /* 以下3种类型、只有当插入对象ID 为空，才自动填充。 */
    /**
     * 全局唯一ID (idWorker)
     */
    ID_WORKER(3),
    /**
     * 全局唯一ID (UUID)
     */
    UUID(4),
    /**
     * 字符串全局唯一ID (idWorker 的字符串表示)
     */
    ID_WORKER_STR(5);

    private int key;

    IdType(int key) {
        this.key = key;
    }
}
```

## 二、自动填充

项目中经常会遇到一些数据，每次都使用相同的方式填充，例如记录的创建时间，更新时间等

1. **实体类上添加注解**

   ```java
   @TableField(fill = FieldFill.INSERT_UPDATE)
   ```

2. **实现元对象处理器接口，重写对应方法**

   ```java
   @Component
   public class MyMetaObjectHandler implements MetaObjectHandler {
       @Override
       public void insertFill(MetaObject metaObject) {
           ...
       }
       @Override
       public void updateFill(MetaObject metaObject) {
           ...
       }
   }
   ```

## 三、乐观锁

当要更新一条数据的时候，希望实现线程安全

### 丢失更新

多个人同时修改同一条记录，最后提交事务，会把之前的提交数据覆盖

### 实现原理

1. 取出记录时，获取当前version
2. 更新时，带上此version，set version = newVersion where version = oldVersion
3. 如果version不对，就更新失败

### 实现步骤

1. 数据库中添加version字段
2. 实体类添加version字段，并添加 @Version 注解
3. 元对象处理器接口添加version的insert默认值
4. 在 MybatisPlusConfig 中注册 Bean

## 四、select

1. selectById
2. selectBatchIds
3. selectByMap

## 五、分页

1. 将PaginationInterceptor插件注入容器

2. 最终通过page对象获取相关数据

   ```java
   //传入两个参数：当前页 和 每页显示记录数
   Page<User> page = new Page<>(1,3);
   //把分页所有数据封装到page对象里面
   userMapper.selectPage(page,null);
   ```

## 六、delete

1. deleteById
2. deleteBatchIds
3. deleteByMap

### 逻辑删除

1. 数据库中添加 deleted字段
2. 实体类添加deleted 字段，并加上 @TableLogic 注解 和 @TableField(fill = FieldFill.INSERT) 注解

3. 元对象处理器接口添加deleted的insert默认值
4. application.properties 加入配置（一般使用默认即可）
5. 将ISqlInjector注入容器

## 七、性能分析

性能分析拦截器，用于输出每条 SQL 语句及其执行时间

SQL 性能执行分析,开发环境使用，超过指定时间，停止运行。有助于发现问题

1. 将PerformanceInterceptor注入容器
2. Spring Boot 中设置dev环境

## 八、复杂条件查询Wrapper

![](..\imgs\mybatis-wrapper.png)

Wrapper ： 条件构造抽象类，最顶端父类

  AbstractWrapper ： 用于查询条件封装，生成 sql 的 where 条件

​    **QueryWrapper** ： Entity 对象封装操作类，不是用lambda语法

​    UpdateWrapper ： Update 条件封装，用于Entity对象更新操作

  AbstractLambdaWrapper ： Lambda 语法使用 Wrapper统一处理解析 lambda 获取 column。

​    LambdaQueryWrapper ：看名称也能明白就是用于Lambda语法使用的查询Wrapper

​    LambdaUpdateWrapper ： Lambda 更新封装Wrapper

### Wrapper中的方法

	1. ge，gt，le，lt：>=， >， <=， <
 	2. eq ，ne：  =， !=(<>)

 	3. like
 	4. orderByDesc，orderByAsc
 	5. last： 拼接sql语句最后