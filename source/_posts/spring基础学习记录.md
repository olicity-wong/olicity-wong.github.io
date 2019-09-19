---
title: spring基础学习记录
tags:
  - java
  - spring
abbrlink: db474090
date: 2018-09-11 21:22:50
---
&emsp;最近在做后端的开发，用到了spring，借这个机会给spring捋一遍吧。东西有点多，需要点时间，抽空学。记录(一)主要是写一点spring最基础的IOC和AOP,至于spring的深入学习、各种注解和SpingMVC另开贴。  
<!--more-->
## 一、Spring概述
### 1.简介
&emsp;Spring是一个基于**控制反转（IOC）**和**面向切面（AOP）**的结构J2EE系统的框架。从简单性、可测试性和松耦合的角度而言，任何Java应用都可以从Spring中受益。  
&emsp;控制反转(IOC)是一个通用概念，Spring最认同的技术是控制反转的**依赖注入（DI）**模式。简单地说就是拿到的对象的属性，已经被注入好相关值了，直接使用即可。
&emsp;面向切面编程（AOP），一个程序中跨越多个点的功能被称为横切关注点，这些横切关注点在概念上独立于应用程序的业务逻辑。Spring 框架的 AOP 模块提供了面向方面的程序设计实现，可以定义诸如方法拦截器和切入点等，从而使实现功能的代码彻底的解耦出来。  
### 2.Spring关键策略
```
* 基于POJO的轻量级和最小侵入性编程
* 通过依赖注入和面向接口实现松耦合
* 基于切面和惯例进行声明式编程
* 通过切面和模板减少样板式代码
```
### 3.Spring优点
```
*方便解耦，简化开发 （高内聚低耦合） 
    Spring就是一个大工厂（容器），可以将所有对象创建和依赖关系维护，交给Spring管理 
    spring工厂是用于生成bean
*AOP编程的支持 
    Spring提供面向切面编程，可以方便的实现对程序进行权限拦截、运行监控等功能
    声明式事务的支持 
    只需要通过配置就可以完成对事务的管理，而无需手动编程
*方便程序的测试 
    Spring对Junit4支持，可以通过注解方便的测试Spring程序
*方便集成各种优秀框架 
    Spring不排斥各种优秀的开源框架，其内部提供了对各种优秀框架（如：Struts、Hibernate、MyBatis、Quartz等）的直接支持
*降低JavaEE API的使用难度 
    Spring 对JavaEE开发中非常难用的一些API（JDBC、JavaMail、远程调用等），都提供了封装，使这些API应用难度大大降低
```
## 二、控制反转（IOC）
### 1.概述
&emsp;代码之间的**耦合性**具有两面性。耦合是必须的，但应小心谨慎的管理。  
&emsp;DI会将所以来的关系自动交给目标对象，而不是让对象自己去获取依赖。–**松耦合**
### 2.获取对象的方式进行比较
传统方式：
&emsp;通过new关键字主动创建一个对象。  
IOC方式：
&emsp;对象的生命周期由Spring来管理，直接从Spring那里去获取一个对象。 IOC是反转控制 (Inversion Of Control)的缩写，就像控制权从本来在自己手里，交给了Spring。  
### 3.装配
&emsp;创建应用组件之间协作的行为通常称为装配。常用XML装配方式。Spring配置文件。
```
*使用构造器进行配置
    <bean> 定义JavaBean，配置需要创建的对象
    id/name ：用于之后从spring容器获得实例时使用的
    class ：需要创建实例的全限定类名


    <bean id="c" class="pojo.Category">
        <property name="name" value="category 1" /> <!--category对象-->
    </bean>

    <!--创建product对象的时候注入了一个category对象，使用ref-->
    <bean name="p" class="pojo.Product"> <!--product类中添加category对象的set&get方法-->
        <property name="name" value="product1" />   <!--product对象-->
        <property name="category" ref="c" />    <!--为product对象注入category对象-->
    </bean>

*使用注解的方式进行配置
    xml中添加  <context:annotation-config/> 注释掉注入category
        *@Autowired注解方法
            在Product.java的category属性前加上@Autowired注解
            @Autowired
            private Category category;
            或者
            @Autowired
            public void setCategory(Category category) 
        *@Resource
            @Resource(name="c")
            private Category category;

*对bean进行注解配置
    直接xml中添加  <context:component-scan base-package="pojo"/>
    就是告知spring，所有的bean都在pojo包下
    通过@Component注解
        在类前加入  @Component("c")  表明此类是bean
        因为配置从applicationContext.xml中移出来了，所以属性初始化放在属性声明上进行了。
        private String name="product 1";
        private String name="category 1";
        同时@Autowired也需要在Product.java的category属性前加上

*Spring javaConfig 后期补充
```
[样例源码参照](https://github.com/olicity-wong/Spring/tree/master/spring)  
## 三、面向切面（AOP）
### 1.概述
&emsp;AOP 即 Aspect Oriented Program 面向切面编程。在面向切面编程的思想里面，把功能分为**核心业务功能**和**周边功能**。  
&emsp;核心业务，比如登陆，增加数据，删除数据都叫核心业务。  
&emsp;周边功能，比如性能统计，日志，事务管理等等。  
&emsp;周边功能在Spring的面向切面编程AOP思想里，即被定义为**切面**  
&emsp;在面向切面编程AOP的思想里面，核心业务功能和切面功能分别独立进行开发，然后把切面功能和核心业务功能 “编织” 在一起，这种能够选择性的，低耦合的把切面和核心业务功能结合在一起的编程思想，就叫AOP。  
### 2.AOP核心概念
```
1、横切关注点
对哪些方法进行拦截，拦截后怎么处理，这些关注点称之为横切关注点
2、切面（aspect）
类是对物体特征的抽象，切面就是对横切关注点的抽象
3、连接点（joinpoint）
被拦截到的点，因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器
4、切入点（pointcut）
对连接点进行拦截的定义
5、通知（advice）
所谓通知指的就是指拦截到连接点之后要执行的代码，通知分为前置、后置、异常、最终、环绕通知五类
6、目标对象
代理的目标对象
7、织入（weave）
将切面应用到目标对象并导致代理对象创建的过程
8、引入（introduction）
在不修改代码的前提下，引入可以在运行期为类动态地添加一些方法或字段
```
### 3.Spring对AOP的支持
&emsp;Spring中AOP代理由Spring的IOC容器负责生成、管理，其依赖关系也由IOC容器负责管理。因此，AOP代理可以直接使用容器中的其它bean实例作为目标，这种关系可由IOC容器的依赖注入提供。Spring创建代理的规则为：  
```
1、默认使用Java动态代理来创建AOP代理，这样就可以为任何接口实例创建代理了
2、当需要代理的类不是代理接口的时候，Spring会切换为使用CGLIB代理，也可强制使用CGLIB
```
### 4.AOP编程
```
1、定义普通业务组件
2、定义切入点，一个切入点可能横切多个业务组件 
3、定义增强处理，增强处理就是在AOP框架为普通业务组件织入的处理动作
```
### 5.简单例子熟悉AOP
```
AOP配置文件
<!--声明业务对象-->
    <bean name="s" class="service.ProductService">
    </bean>
    <!--声明日志切面-->
    <bean id="loggerAspect" class="aspect.LoggerAspect"/>
    <aop:config>
        <!--指定核心业务功能-->
        <aop:pointcut id="loggerCutpoint"
                      expression=
                              "execution(* service.ProductService.*(..)) "/>    <!--* 返回任意类型,   包名以 service.ProductService 开头的类的任意方法,   (..) 参数是任意数量和类型-->
        <!--指定辅助业务功能-->
        <aop:aspect id="logAspect" ref="loggerAspect">
            <aop:around pointcut-ref="loggerCutpoint" method="log"/>
        </aop:aspect>
    </aop:config>
```
### 6.注解方式AOP
```
1.注解注解配置业务类,使用@Component("s")
2.注解配置切面
    @Aspect 注解表示这是一个切面
    @Component 表示这是一个bean,由Spring进行管理
    @Around(value = "execution(* service.ProductService.*(..))") 表示对service.ProductService 这个类中的所有方法进行切面操作
3.配置applicationContext.xml
    <!--扫描这俩包，定位业务类和切面类-->
    <context:component-scan base-package="aspect"/>
    <context:component-scan base-package="service"/>
    <!--找到被注解了的切面类，进行切面配置-->
    <aop:aspectj-autoproxy/>
```
[样例源码参照](https://github.com/olicity-wong/Spring)
## 四、总结说明
&emsp;这个帖子主要就是简单的介绍了一下spring的IOC和AOP思想以及简单的例子。暂时写这么多，spring东西比较多，下一阶段应该是再看一下Spring各种注解的使用还有SpringMVC。