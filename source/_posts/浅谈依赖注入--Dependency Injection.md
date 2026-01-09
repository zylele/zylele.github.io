---
title: 浅谈依赖注入--Dependency Injection
date: 2015-05-16 18:24:15
tags: 
  - Spring
  - 依赖注入
---

## 概述

**Spring**能有效地组织J2EE应用各层的对象。不管是控制层的Action对象，还是业务层的Service对象，还是持久层的DAO对象，都可在Spring的 管理下有机地协调、运行。Spring将各层的对象以`松耦合`的方式组织在一起，Action对象无须关心Service对象的具体实现，Service对 象无须关心持久层对象的具体实现，各层对象的调用`完全面向接口`。当系统需要重构时，代码的改写量将大大减少。

<!-- more -->

上面所说的一切都得宜于Spring的核心机制，`依赖注入`。

> 依赖注入让bean与bean之间以配置文件组织在一起，而不是以硬编码的方式耦合在一起。

依赖注入(Dependency Injection)和控制反转(Inversion of Control)是同一个概念。具体含义是:

> 当某个角色(可能是一个Java实例，调用者)需要另一个角色(另一个Java实例，被调用者)的协助时，在传统的程序设计过程中，通常由调用者来创建被调用者的实例。但在Spring里，创建被调用者的工作不再由调用者来完成，因此称为控制反转;创建被调用者实例的工作通常由Spring容器来完成，然后注入调用者，因此也称为依赖注入。

不管是依赖注入，还是控制反转，都说明Spring采用动态、灵活的方式来管理各种对象。对象与对象之间的具体实现互相透明。在理解依赖注入之前，看如下这个问题在各种社会形态里如何解决:

**`一个人`(Java实例，调用者)需要一把`斧子`(Java实例，被调用者)**

*   原始社会里，几乎没有社会分工。需要斧子的人(调用者)只能自己去磨一把斧子(被调用者)。对应的情形为:Java程序里的调用者自己创建被调用者。

*   进入工业社会，工厂出现。斧子不再由普通人完成，而在工厂里被生产出来，此时需要斧子的人(调用者)找到工厂，购买斧子，无须关心斧子的制造过程。对应Java程序的简单工厂的设计模式。

*   进入“按需分配”社会，需要斧子的人不需要找到工厂，坐在家里发出一个简单指令:需要斧子。斧子就自然出现在他面前。对应Spring的依赖注入。

_第一种情况下，Java实例的调用者创建被调用的Java实例，必然要求被调用的Java类出现在调用者的代码里。无法实现二者之间的松耦合。_

_第二种情况下，调用者无须关心被调用者具体实现过程，只需要找到符合某种标准(接口)的实例，即可使用。此时调用的代码面向接口编程，可以让调用者和被调用者解耦，这也是工厂模式大量使用的原因。但调用者需要自己定位工厂，调用者与特定工厂耦合在一起。_

_第三种情况下，调用者无须自己定位工厂，程序运行到需要被调用者时，系统自动提供被调用者实例。事实上，调用者和被调用者都处于Spring的管理下，二者之间的依赖关系由Spring提供。_

* * *

## 依赖注入方式

所谓依赖注入，是指程序运行过程中，如果需要调用另一个对象协助时，无须在代码中创建被调用者，而是依赖于外部的注入。Spring的依赖注入对调用者和被调用者几乎没有任何要求，完全支持对POJO之间依赖关系的管理。依赖注入通常有两种:

*   设值注入。

*   构造注入。

### 设值注入

设值注入是指通过setter方法传入被调用者的实例。这种注入方式简单、直观，因而在Spring的依赖注入里大量使用。看下面代码，是Person的接口

```java
//定义Person接口
public interface Person{
    //Person接口里定义一个使用斧子的方法
    public void useAxe();
}

```

然后是Axe的接口

```java
//定义Axe接口
public interface Axe{
    //Axe接口里有个砍的方法
    public void chop();
}

```

Person的实现类

```java
//Chinese实现Person接口
public class Chinese implements Person{
    //面向Axe接口编程，而不是具体的实现类
    private Axe axe;
    //默认的构造器
    public Chinese(){}
    //设值注入所需的setter方法
    public void setAxe(Axe axe){
        this.axe = axe;
    }
    //实现Person接口的useAxe方法
    public void useAxe(){
        System.out.println(axe.chop());
    }
}

```

Axe的第一个实现类

```java
//Axe的第一个实现类 StoneAxe
public class StoneAxe implements Axe{
    //默认构造器
    public StoneAxe(){}
    //实现Axe接口的chop方法
    public String chop(){
        return "石斧砍柴好慢";
    }
}

```

下面采用Spring的配置文件将Person实例和Axe实例组织在一起。配置文件如下所示:

```xml
<!-- Spring配置文件的根元素 -->
<BEANS>
<!—定义第一bean，该bean的id是chinese, class指定该bean实例的实现类 -->
<BEAN class=lee.Chinese id=chinese>
<!-- property元素用来指定需要容器注入的属性，axe属性需要容器注入此处是设值注入，因此Chinese类必须拥有setAxe方法 -->
<property name="axe">
<!-- 此处将另一个bean的引用注入给chinese bean -->
<REF local="”stoneAxe”/">
</property>
</BEAN>
<!-- 定义stoneAxe bean -->
<BEAN class=lee.StoneAxe id=stoneAxe />
</BEANS>

```

从配置文件中，可以看到Spring管理bean的灵巧性。bean与bean之间的依赖关系放在配置文件里组织，而不是写在代码里。通过配置文件的 指定，Spring能精确地为每个bean注入属性。因此，配置文件里的bean的class元素，不能仅仅是接口，而必须是真正的实现类。

Spring会自动接管每个bean定义里的property元素定义。Spring会在执行无参数的构造器后、创建默认的bean实例后，调用对应 的setter方法为程序注入属性值。property定义的属性值将不再由该bean来主动创建、管理，而改为被动接收Spring的注入。

每个bean的id属性是该bean的惟一标识，程序通过id属性访问bean，bean与bean的依赖关系也通过id属性完成。

下面看主程序部分:

```java
public class BeanTest{
    public static void main(String[] args)throws Exception{
        //因为是独立的应用程序，显式地实例化Spring的上下文。
        ApplicationContext ctx = new FileSystemXmlApplicationContext("bean.xml");
        //通过Person bean的id来获取bean实例，面向接口编程，因此此处强制类型转换为接口类型
        Person p = (Person)ctx.getBean("chinese");
        //直接执行Person的userAxe()方法。
        p.useAxe();
    }
}

```

程序的执行结果如下:

```java
石斧砍柴好慢

```

主程序调用Person的useAxe()方法时，该方法的方法体内需要使用Axe的实例，但程序里`没有任何地方`将特定的Person实例和Axe实例`耦合`在一起。或者说，程序里没有为Person实例传入Axe的实例，Axe实例由Spring在运行期间动态注入。

Person实例不仅不需要了解Axe实例的具体实现，甚至无须了解Axe的创建过程。程序在运行到需要Axe实例的时候，Spring创建了Axe 实例，然后注入给需要Axe实例的调用者。Person实例运行到需要Axe实例的地方，自然就产生了Axe实例，用来供Person实例使用。

调用者不仅`无须关心被调用者的实现过程`，连工厂定位都可以省略(~~真的是按需分配啊!~~)。

如果需要改写Axe的实现类。或者说，提供另一个实现类给Person实例使用。Person接口、Chinese类都无须改变。只需提供另一个Axe的实现，然后对配置文件进行简单的修改即可。

Axe的另一个实现如下:

```java
//Axe的另一个实现类 SteelAxe
public class SteelAxe implements Axe{
    //默认构造器
    public SteelAxe(){}
    //实现Axe接口的chop方法
    public String chop(){
        return "钢斧砍柴真快";
    }
}

```

然后，修改原来的Spring配置文件，在其中增加如下一行:

```xml
<!-- 定义一个steelAxe bean-->
<BEAN class=lee.SteelAxe id=steelAxe />

```

该行重新定义了一个Axe的实现:SteelAxe。然后修改chinese bean的配置，将原来传入stoneAxe的地方改为传入steelAxe。也就是将

```xml
<REF local="”stoneAxe”/">

```

改成

```xml
<REF local="”steelAxe”/">

```

此时再次执行程序，将得到如下结果:

```java
钢斧砍柴真快

```

Person与Axe之间没有任何代码耦合关系，bean与bean之间的依赖关系由Spring管理。采用setter方法为目标bean注入属性的方式，称为设值注入。

业务对象的更换变得相当简单，对象与对象之间的依赖关系从代码里分离出来，通过配置文件动态管理。

### 构造注入

所谓构造注入，指通过构造函数来完成依赖关系的设定，而不是通过setter方法。对前面代码Chinese类做简单的修改，修改后的代码如下:

```java
//Chinese实现Person接口
public class Chinese implements Person{
    //面向Axe接口编程，而不是具体的实现类
    private Axe axe;
    //默认的构造器
    public Chinese(){}
    //构造注入所需的带参数的构造器
    public Chinse(Axe axe){
        this.axe = axe;
    }
    //实现Person接口的useAxe方法
    public void useAxe(){
        System.out.println(axe.chop());
    }
}

```

此时无须Chinese类里的setAxe方法，构造Person实例时，Spring为Person实例注入所依赖的Axe实例。构造注入的配置文件也需做简单的修改，修改后的配置文件如下:

```xml
<!-- Spring配置文件的根元素 -->
<BEANS>
<!—定义第一个bean，该bean的id是chinese, class指定该bean实例的实现类 -->
<BEAN class=lee.Chinese id=chinese>
</BEAN>
<!-- 定义stoneAxe bean -->
<BEAN class=lee.SteelAxe id=steelAxe />
</BEANS>

```

执行效果与使用steelAxe设值注入时的执行效果`完全一样`。区别在于:创建Person实例中Axe属性的`时机`不同——设值注入是现创建一个默认的bean实例，然后调用对应的构造方法注入依赖关系。而构造注入则在创建bean实例时，已经完成了依赖关系的。