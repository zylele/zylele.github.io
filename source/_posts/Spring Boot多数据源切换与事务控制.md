---
title: Spring Boot多数据源切换与事务控制
date: 2016-09-04 18:25:22
tags: 
  - Spring Boot
  - 事务
---

后台数据库读写分离，不光是要配置多个数据源，还得能灵活动态的切换数据源，很好，目前都没问题，然而如果你的应用是使用SpringBoot：

> SpringBoot使我们更容易去创建基于Spring的独立和产品级的可以“即时运行”的应用和服务。支持约定大于配置，目的是尽可能快地构建和运行Spring应用。

来初始化构建你的工程，引入多数据源将可能会导致事务无效的问题`本文重点`。因为传统通过xml手动配置更精准，出错也容易查找原因，然而交给SpringBoot自动帮你完成大部分的配置，绝逼满满的都是坑(我的直觉

<!-- more -->

好，以下正题。

(本文持久层框架使用MyBatis)

*   简单的架构是：单个数据源绑定给sessionFactory，再在Dao层操作

![](https://zylele.github.io/img/sprinboot-datasources/datasource.jpg)

*   若多个数据源的话

![](https://zylele.github.io/img/sprinboot-datasources/datasources.jpg)

*   sessionFactory都写死在了Dao层，若我再添加个数据源的话，则又得添加一个sessionFactory，这样并不能扩展嘛，所以

![](https://zylele.github.io/img/sprinboot-datasources/dynamic-datasources.jpg)

这样才是坠吼的！

## 多数据源实现原理：

### 配置文件

精简篇幅，省略了无关本内容主题的配置

本工程关于数据源的配置在pom.xml，部署各种环境应用不同的数据源，测试两个数据库test1和test2的配置

```xml
<properties>
    <!-- 驱动 -->
    <master.jdbc.driver>com.mysql.jdbc.Driver</master.jdbc.driver>
    <!-- JDBC URL -->
    <master.jdbc.url>jdbc:mysql://localhost/test1?useUnicode=true&amp;autoReconnect=true</master.jdbc.url>
    <!-- 数据库用户名 -->
    <master.jdbc.username>root</master.jdbc.username>
    <!-- 数据库密码 -->
    <master.jdbc.password>root</master.jdbc.password>
    <!-- 连接池最小连接数 -->
    <master.db.pool.min>10</master.db.pool.min>
    <!-- 连接池初始连接数 -->
    <master.db.pool.init>10</master.db.pool.init>
    <!-- 连接池最大连接数 -->
    <master.db.pool.max>20</master.db.pool.max>
    <!-- 驱动 -->
    <slave.jdbc.driver>com.mysql.jdbc.Driver</slave.jdbc.driver>
    <!-- JDBC URL -->
    <slave.jdbc.url>jdbc:mysql://localhost/test2?useUnicode=true&amp;autoReconnect=true</slave.jdbc.url>
    <!-- 数据库用户名 -->
    <slave.jdbc.username>root</slave.jdbc.username>
    <!-- 数据库密码 -->
    <slave.jdbc.password>root</slave.jdbc.password>
    <!-- 连接池最小连接数 -->
    <slave.db.pool.min>10</slave.db.pool.min>
    <!-- 连接池初始连接数 -->
    <slave.db.pool.init>10</slave.db.pool.init>
    <!-- 连接池最大连接数 -->
    <slave.db.pool.max>20</slave.db.pool.max>
</properties>

```

application.properties，由SpringBoot自动加载相关属性

```
master.datasource.name=masterDataSource
master.datasource.url=@master.jdbc.url@
master.datasource.username=@master.jdbc.username@
master.datasource.password=@master.jdbc.password@
master.datasource.driver-class-name=@master.jdbc.driver@
master.datasource.max-idle=@master.db.pool.max@
master.datasource.min-idle=@master.db.pool.min@
master.datasource.initial-size=@master.db.pool.init@
master.datasource.validation-query=select 1
master.datasource.test-on-borrow=true
master.datasource.test-while-idle=true
slave.datasource.name=slaveDataSource
slave.datasource.url=@slave.jdbc.url@
slave.datasource.username=@slave.jdbc.username@
slave.datasource.password=@slave.jdbc.password@
slave.datasource.driver-class-name=@slave.jdbc.driver@
slave.datasource.max-idle=@slave.db.pool.max@
slave.datasource.min-idle=@slave.db.pool.min@
slave.datasource.initial-size=@slave.db.pool.init@
slave.datasource.validation-query=select 1
slave.datasource.test-on-borrow=true
slave.datasource.test-while-idle=true
```

transaction.xml配置需要事务控制的service

```xml
<!-- AOP配置-->
<aop:config>
    <!--pointcut元素定义一个切入点，execution中的第一个星号 用以匹配方法的返回类型，这里星号表明匹配所有返回类型。
    com.abc.dao.*.*(..)表明匹配com.abc.dao包下的所有类的所有方法-->
    <aop:pointcut id="transactionPointCut" expression="execution(* cn.abc.lele.*.service.impl..*.*(..))" />
    <!--将定义好的事务处理策略应用到上述的切入点-->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="transactionPointCut" />
</aop:config>
<tx:advice id="txAdvice" transaction-manager="transactionManager" >
    <tx:attributes>
    <!--所有以find开头的方法都是只读的-->
    <tx:method name="query*" read-only="true" />
    <tx:method name="select*" read-only="true" />
    <tx:method name="get*" read-only="true" />
    <tx:method name="is*" read-only="true" />
    <tx:method name="find*" read-only="true" />
    <tx:method name="fill*" read-only="true" />
    <tx:method name="count*" read-only="true" />
    <tx:method name="add*" />
    <tx:method name="insert*" />
    <tx:method name="save*" />
    <tx:method name="update*"/>
    <tx:method name="change*" />
    <tx:method name="delete*" />
    <tx:method name="remove*" />
    <tx:method name="clean*" />
    <tx:method name="active*" />
    <tx:method name="deactive*" />
    <tx:method name="enable*"/>
    <tx:method name="disable*" />
    <tx:method name="accept*" />
    <!--其他方法使用默认事务策略 propagation="NEVER" -->
    <tx:method name="*"  propagation="NEVER"/>
    </tx:attributes>
</tx:advice>

```

### 扩展Spring的AbstractRoutingDataSource抽象类（该类充当了DataSource的路由中介, 能有在运行时, 根据某种key值来动态切换到真正的DataSource上。）

查看源码，AbstractRoutingDataSource的声明

```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean
```

它继承了AbstractDataSource，而AbstractDataSource是javax.sql.DataSource的子类，分析它的getConnection方法

```java
public Connection getConnection() throws SQLException {
    return determineTargetDataSource().getConnection();  
}
public Connection getConnection(String username, String password) throws SQLException {  
    return determineTargetDataSource().getConnection(username, password);  
}
```

再查看determineTargetDataSource()方法

```java
protected DataSource determineTargetDataSource() {
    Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");  
    Object lookupKey = determineCurrentLookupKey();
    DataSource dataSource = this.resolvedDataSources.get(lookupKey);
    if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
        dataSource = this.resolvedDefaultDataSource;
    }
    if (dataSource == null) {
        throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
    }
    return dataSource;
}
```

这里的重点是determineCurrentLookupKey()方法，这是AbstractRoutingDataSource类中的一个抽象方法，而它的返回值是你所要用的数据源dataSource的key值，有了这个key值，resolvedDataSource（这是个map,由配置文件中设置好后存入的）就从中取出对应的DataSource，如果找不到，就用配置默认的数据源

没错，要扩展AbstractRoutingDataSource类，并重写其中的determineCurrentLookupKey()方法，来实现数据源的切换

```java
public class ReadWriteSplitRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DbContextHolder.getDbType();
    }
}

```

DbContextHolder是我们封装的对数据源进行操作的类

```java
public class DbContextHolder {
    public enum DbType {
        MASTER,
        SLAVE
    }
    private static final ThreadLocal<DbType> contextHolder = new ThreadLocal<DbType>();
    public static void setDbType(DbType dbType) {
        if(dbType == null){
            throw new NullPointerException();
        }
        contextHolder.set(dbType);
    }
    public static DbType getDbType() {
        return contextHolder.get() == null ? DbType.MASTER : contextHolder.get();
    }
    public static void clearDbType() {
        contextHolder.remove();
    }
}
```

这里的setDbType()什么时候执行呢？当然是在需要切换数据源的时候执行，应用面向切面，增加一个注解标签，在service层中需要切换数据源的方法上，写上注解标签，调用相应方法切换数据源，这里的@ReadOnlyConnection将在service层中切换到读库

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnlyConnection {
}
```

增加@Aspect的一个切面拦截类，切换数据源

```java
@Aspect
@Component
public class ReadOnlyConnectionInterceptor implements Ordered {
    private static final Logger logger = LoggerFactory.getLogger(ReadOnlyConnectionInterceptor.class);
    @Around("@annotation(readOnlyConnection)")
    public Object proceed(ProceedingJoinPoint proceedingJoinPoint, ReadOnlyConnection readOnlyConnection) throws Throwable {
        try {
            logger.info("set database connection to read only");
            DbContextHolder.setDbType(DbContextHolder.DbType.SLAVE);
            Object result = proceedingJoinPoint.proceed();
            return result;
        } finally {
            DbContextHolder.clearDbType();
            logger.info("restore database connection");
        }
    }
    @Override
    public int getOrder() {
        return 0;
    }
}
```

### 数据源加载与事务控制

还记得上面那个由SpringBoot自动加载相关属性的application.properties么

**SpringBoot会自动根据application.properties将数据源属性前缀是spring.datasource配置`单数据源`，并且初始化相应的SqlSessionFactory(数据库session的连接工厂)与TransactionManager(事务管理器)**

这句话是重点，念三遍

所以，在多数据源的需求下，必须要我们手动初始化相应的bean

```java
@Configuration
@EnableAutoConfiguration
@MapperScan(basePackages = "cn.abc.lele.*.mapper", sqlSessionFactoryRef = "sqlSessionFactory")
public class DatabaseConfiguration {
    @Autowired
    private ApplicationContext appContext;
    //初始化主库
    @Bean(name = "masterDataSource")
    @ConfigurationProperties(prefix = "master.datasource")
    @Primary
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }
    //初始化从库
    @Bean(name = "slaveDataSource")
    @ConfigurationProperties(prefix = "slave.datasource")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    //初始化数据源bean，这个bean很重要，后面事务控制也会用到
    @Bean
    public AbstractRoutingDataSource roundRobinDataSouceProxy(@Qualifier("masterDataSource")DataSource master,  @Qualifier("slaveDataSource") DataSource slave) {
        ReadWriteSplitRoutingDataSource proxy = new ReadWriteSplitRoutingDataSource();
        Map<Object, Object> targetDataSources = new HashMap<Object, Object>();
        targetDataSources.put(DbContextHolder.DbType.MASTER, master);
        targetDataSources.put(DbContextHolder.DbType.SLAVE,  slave);
        proxy.setDefaultTargetDataSource(master);
        proxy.setTargetDataSources(targetDataSources);
        return proxy;
    }
    //初始化SqlSessionFactory，将自定义的多数据源ReadWriteSplitRoutingDataSource类实例注入到工厂中
    @Bean
    public SqlSessionFactory sqlSessionFactory(@Qualifier("masterDataSource")DataSource master, @Qualifier("slaveDataSource") DataSource slave) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();       
        sessionFactory.setDataSource((DataSource)appContext.getBean("roundRobinDataSouceProxy"));
        return sessionFactory.getObject();
    }
}
```

到这里，启动工程，多数据源切换能正常执行，但是你会发现事务失效，这是为什么呢？

我们初始化了两个数据源，并且注入给SqlSessionFactory，所以对两个数据源切换并各自访问完全没有问题，让我们回顾一下上面的说过SpringBoot的一个作用：

**SpringBoot会自动根据application.properties将数据源属性前缀是spring.datasource配置`单数据源`，并且初始化相应的SqlSessionFactory(数据库session的连接工厂)与TransactionManager(事务管理器)**

所以，这里SpringBoot即使找到数据源属性前缀spring.datasource的数据源配置，也只是单数据源，这就是为什么多数据源切换正常执行，而事务失效的原因！

因为TransactionManager事务管理器里的dataSource根本不是我们的masterDataSource和slaveDataSource(我觉得应该是null，待验证

所以，必须手动初始化一个多数据源的TransactionManager，并且指定bean的名称与上面的transaction.xml中的`transaction-manager="transactionManager"`一致！这样，Spring将会使用我们初始化之后的TransactionManager。

新增一个MyDataSourceTransactionManagerAutoConfiguration事务管理器，继承SpringBoot的jar包中DataSourceTransactionManagerAutoConfiguration自动配置数据源事务管理器类，并且构造注入我们初始化的数据源ReadWriteSplitRoutingDataSource的实例

```java
@Configuration
@EnableTransactionManagement
public class MyDataSourceTransactionManagerAutoConfiguration extends DataSourceTransactionManagerAutoConfiguration {
    @Autowired
    private ApplicationContext appContext;
    /**
     * 自定义事务
     * MyBatis自动参与到spring事务管理中，无需额外配置，只要org.mybatis.spring.SqlSessionFactoryBean引用的数据源与DataSourceTransactionManager引用的数据源一致即可，否则事务管理会不起作用。
     * @return
     */
    @Bean(name = "transactionManager")
    public DataSourceTransactionManager transactionManagers() {
        return new DataSourceTransactionManager((DataSource)appContext.getBean("roundRobinDataSouceProxy"));
    }
}
```

重新启动工程，事务测试通过。