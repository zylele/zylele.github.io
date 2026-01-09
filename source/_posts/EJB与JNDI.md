---
title: EJB与JNDI
date: 2016-04-04 18:25:10
tags: 
  - EJB
---

<blockquote class="blockquote-center">应用驱动学习</blockquote>
（摊手）

第一次接触使用EJB构建项目部署。

`EJB`(Enterprise JavaBean)是sun的JavaEE服务器端组件模型和“`最佳实践`（讲真，即使EJB3.0已经借鉴了Spring等优点，在我看来还是臃肿繁琐），设计目标与核心应用是部署分布式应用程序。

> 凭借java跨平台的优势，用EJB技术部署的分布式系统可以不限于特定的平台。EJB是J2EE的一部分，定义了一个用于开发基于组件的企业多重应用程序的标准。其特点包括网络服务支持和核心开发工具(SDK)。 在J2EE里，Enterprise Java Beans(EJB)称为Java企业Bean，是Java的核心代码，分别是会话Bean（Session Bean），实体Bean（Entity Bean）和消息驱动Bean（MessageDriven Bean）。

<!-- more -->

作为Java WEB项目构建，可以简单这么理解其工作流程：

**JSP → Servlet → Session Bean或MessageDriven Bean → Entity Bean → DB持久化**

…

（Spring不知道比它高到哪里去了）

`JNDI`(The Java Naming and Directory Interface，Java 命名和目录接口) 是一组在Java 应用中访问命名和目录服务的API。为开发人员提供了查找和访问各种命名和目录服务的通用、统一的方式。借助于JNDI 提供的接口，能够通过名字定位用户、机器、网络、对象服务等。

与EJB通信必须借助JNDI的命名服务查找使用相关的Bean，其中：

local是本地接口，remote是远程接口。web层调用app层使用remote接口。session bena和entity bean之间
调用使用的是local接口。

因为JNDI是一组接口，所以我们只需根据接口规范编程就可以。要通过JNDI 进行资源访问，我们必须设置初始化上下文的参数，主要是设置JNDI 驱动的类名(java.naming.factory.initial) 和提供命名服务的URL (java.naming.provider.url)。

因为Jndi 的实现产品有很多。所以java.naming.factory.initial 的值因提供JNDI 服务器的不同而不同，java.naming.provider.url 的值包括提供命名服务的主机地址和端口号。

访问Jboss 服务器的例子代码：

```java
Properties props = new Properties();
props.setProperty("java.naming.factory.initial", "org.jnp.interfaces.NamingContextFactory");
props.setProperty("java.naming.provider.url", "localhost:1099");
InitialContext ctx = new InitialContext(props);
HelloWorld helloworld = (HelloWorld) ctx.lookup("HelloWorldBean/remote");

```

不用说你也明白，remote接口对性能的影响是很大的。所以在设计的时候我们应尽量使用local接口，也就是facade模式。具体来说是，web层调用app层的session bean,session bean在调用各个实体entity bean。

local接口可以在与ejb同一个jvm环境中调用，但是不能对它进行远程调用的，在jndi查找的时候不能查找local home，而要查找remote home，也就是说需要实际进行RMI调用，而且必须提供Provider URL(例如t3://myserver:7001)，而且他们返回给客户的对象也不一样，local home创建的是javax.ejb.EJBLocalObject类型，它没有继承Remote interface；而Remote home创建的是javax.ejb.EJBObject类型的，它扩展了Remote。

实际上javax.ejb.EJBLoclObject型接口没有抛出RemoteException，因为对local类型接口的调用不是RMI，所以对Local接口的调用效率要高于对remote对象的调用，针对这点对EJB的设计提出以下建议：

*   如果你的ejb客户只可能存在于与ejb相同app server，或者说同一个JVM环境中时，你可以只生成local类型接口（包括EJBHome 与EJBObject），如果你需要在与EJB容器不同的JVM环境中调用你的EJB的话，你必须生成Remote类型的接口（包括EJBHome 与EJBObject）。

*   在一般情况下建议两种类型的接口（包括EJBHome 与EJBObject）都生成，尤其是Session Bean，Entity Bean，可以只生成local类型的接口，如果想远程调用你的Entity Bean一般用Session Bean做代理。

*   如果你不是远程调用EJB的话，使用EJB时建议调用local接口，这样效率高，因为远程调用就意味着建立网络连接，效率必然不如local调用。

*   在Java EE 7中设计EJB时，默认情况下只给你生成local类型接口，所以你需要在设计EJB时把interfaces设成：local/remote，这样的话你的EJB至少会有5个java文件