---
title: Dubbox快速指南
date: 2017-03-07 21:44:05
tags: 
  - Dubbox
---

该demo的github地址：[dubbox-demo](https://github.com/zylele/dubbox-demo)

# 准备

## 依赖

建议在maven安装目录下的`conf\settings.xml`的`<mirrors>`标签中添加如下镜像，以提高maven打包速度

```xml
<mirror>
    <id>alimaven</id>
    <name>aliyun maven</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    <mirrorOf>central</mirrorOf>
</mirror>
```

下载dubbox源码并且编译：<!-- more -->

> git clone git@github.com:dangdangdotcom/dubbox.git
执行maven命令`mvn clean install -Dmaven.test.skip`

## 注册中心

推荐使用Zookeeper注册中心

[zookeeper安装](http://dubbo.io/Administrator+Guide.htm#AdministratorGuide-ZookeeperRegistryInstallation)

Zookeeper是Apacahe Hadoop的子项目，是一个树型的目录服务，支持变更推送，适合作为Dubbo服务的注册中心，工业强度较高，可用于生产环境，并推荐使用，参见：[Apache ZooKeeper](http://zookeeper.apache.org)

## 接口定义与通用实体

common工程只定义了服务提供者与消费者所依赖的接口与实体类

该工程下的接口与实体在服务提供方和消费方共享

## 配置管理

config工程只负责管理dubbo通用配置，在服务提供方和消费方共享，除此之外也可配置如数据库、缓存、队列等等

# 快速启动

## 属性配置

执行初始化init.sql

该demo基于springboot，以properties方式配置公共部分，xml配置各个服务不同之处

config工程引入依赖所需的jar

application.properties中加入注册中心配置：

```
dubbo.registry.address=@dubbo.registry.address@
```

具体值请修改pom不同环境打包的profile标签指定

config工程另外配置数据库，缓存等等，这里不再赘述

## 服务提供者

引入common与config

新建dubbo-config.xml,更多配置详情请参考[dubbo配置参考手册](http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-%E9%85%8D%E7%BD%AE%E5%8F%82%E8%80%83%E6%89%8B%E5%86%8C)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
    http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="base-user-service" />

    <!-- 使用zookeeper注册中心暴露服务地址 -->
    <dubbo:registry address="${dubbo.registry.address}" file="c:/dubbo/base-user-service.cache"/>

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    
    <!-- 通过注册中心发现监控中心服务 -->
    <dubbo:monitor protocol="registry" />
    
    <!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
    <dubbo:annotation package="cn.zylele.base" />
    
    <!-- 服务提供者缺省值配置 -->
    <dubbo:provider timeout="60000" delay="-1" retries="0" />
    
</beans>
```

服务提供者实现服务接口，这里的@Service的注解是dubbo的服务提供方注解，声明需要暴露的服务接口，并指定实现

注：服务既可以是提供者也可以是消费者，下面的DynamicQueryService为另一个服务。

`@Reference`是dubbo的服务消费方注解，生成远程服务代理，来引用别的服务

``` java
@Component
@Service
public class UserQueryServiceImpl implements UserQueryService{

    @Reference(check = false)
    DynamicQueryService dynamicQueryService;
    
    @Autowired
    UserMapper userMapper;
    
    @Override
    public User getUser(String userid) {
        return userMapper.getUser(userid);
    }
}
```

启动入口，发布服务

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;

@Configuration
@ImportResource(value = "classpath:*.xml")
@SpringBootApplication
public class UserServiceProvider {
    
    public static void main(String[] args) throws InterruptedException {

        SpringApplication app = new SpringApplication(UserServiceProvider.class);
        app.setWebEnvironment(false);
        app.run(args);
    }
}
```

## 服务消费者

引入common

application.properties中加入注册中心配置：

```
dubbo.registry.address=@dubbo.registry.address@
```

具体值请修改pom不同环境打包的profile标签指定

新建dubbo-config.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
    http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
    <dubbo:application name="base-consumer" />

    <!-- 使用zookeeper注册中心暴露发现服务地址 -->
    <dubbo:registry address="${dubbo.registry.address}" file="c:/dubbo/base-consumer.cache"/>

    <!-- 通过注册中心发现监控中心服务 -->
    <dubbo:monitor protocol="registry" />
    
    <!-- 扫描注解包路径，多个包用逗号分隔，不填pacakge表示扫描当前ApplicationContext中所有的类 -->
    <dubbo:annotation package="cn.zylele.base" />
    
    <!-- 关闭所有服务的启动时检查 check=false，总是会返回引用，当服务恢复时，能自动连上-->
    <dubbo:consumer check="false" />
</beans>
```

发现并调用远程服务

```java
@RestController
public class UserController {

    @Reference
    UserQueryService userQueryService;
    
    @RequestMapping(value="/user/get/{userid}",method=RequestMethod.GET, 
        produces = MediaType.APPLICATION_JSON_VALUE)
    public User readUserInfo(@PathVariable("userid") String userid){
        return userQueryService.getUser(userid);
    }
}
```

启动工程，调试

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.embedded.ConfigurableEmbeddedServletContainer;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;

@Configuration
@ImportResource(value = "classpath:dubbo-config.xml")
@SpringBootApplication
public class BaseConsumer implements EmbeddedServletContainerCustomizer{
    public static void main(String[] args) {
        SpringApplication.run(BaseConsumer.class, args);
    }

    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setPort(1000);
    }
}
```

---

dubbox还具有相当多的配置功能，如负载均衡、集群容错，多协议、多注册中心等

更多示例与参考手册，可查看[dubbo用户指南](http://dubbo.io/User+Guide-zh.htm)