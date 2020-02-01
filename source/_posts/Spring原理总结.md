---
title: Spring原理总结
sidebar: [grid, toc, tags] # 放置任何你想要显示的侧边栏部件
categories: [java进阶, java虚拟机]
date: 2019-12-19
tags:
- jvm
- java
- 总结
---



本文不是源码分析，源码分析的过程很复杂，本文的目的是使用精简的语言，基于源码分析的基础，将框架原理进行总结。

##### 1. Spring Boot自动装配的原理

首先在使用Spring Boot的时候main方法所在的类会加上一个叫做`@SpringBootApplication`的注解，这个注解之上主要有三个注解

- `@SpringBootConfiguration`

- `@EnableAutoConfiguratio`

- `@ComponentScan`

其中`@SpringBootConfiguratio`注解被`@Configuration`注解修饰，作用是将main方法所在的那个类声明成一个配置类，然后`@ComponentScan`的作用使容器扫描与main方法同一级的包及其子包，然后最重要的一个注解`@EnableAutoConfiguration`，**这个注解与自动装配密切相关**。

注解`@EnableAutoConfiguration`上面有一个注解`@Import`，这个注解的作用是将一些类注入到容器中，`@Import`注解有三个用法

- 直接将某一个指定的类注入到容器中

- 借助于ImportBeanDefinitionRegistrar接口将类注入到容器中

- 借助于ImportSelector类将类注入到容器中

Spring Boot使用的是第三种方式，借助一个叫`AutoConfigurationImportSelector`的类，返回类的全限定名数组，这些数组中的元素所代表的的类都会被注入到容器中，具体是哪些类呢？首先会去加载所有Spring预先定义的配置条件信息，这些信息位于`org.springframework.boot.autoconfigure`包下的`META-INF/spring-autoconfigure-metadata.properties`文件中，这些类都是被注解`@Configuration`修饰的配置类，这些配置类还存在一些和条件装配相关的注解，配置类中约定好了配置方式，这样用户就不需要手动去配置，如果需要修改模型信息，可以修改yml文件的内容，这也就是**所谓的约定大于配置**。



##### 2. Spring IOC的原理

首先解释一下什么是IOC，IOC的意思是控制反转，控制反转是一种思想，它指的是将类管理自身成员变量的权利交给第三方容器，也就是说在没有使用IOC容器的时候，一个对象所依赖的成员变量是需要自己管理、实例化的，但是有了IOC之后，程序员只要通过配置信息描述对象与对象之间的关系，然后交给ioc容器，容器会自动帮我们配置好类与类之间的关系。

然后在Spring中实现控制反转的方式叫做DI，也就是依赖注入。首先我们使用Spring创建IOC容器是通过ApplicationContex这个类创建的，而这个类有一个顶层的类，叫做BeanFactory，BeanFactory这个类不是由用户直接使用的，而是Spring内部的一个很重要的类，它是实现了ioc的基本功能。ApplicationContex也有几个子类，如`ClassPathXmlApplicationContex`、`FileSystemXmlApplicationContex`、`AnnotationConfigApplicationContex`，这些子类的区别在于配置信息的位置和类型不同，比如`ClassPathXmlApplicationContex`的配置信息是XML文件，并且会在ClassPath下找，而`FileSystemXmlApplicationContex`的配置文件是xml文件，但是需要提供一个全路径名的xml文件，`AnnotationConfigApplicationContex`它的配置信息是java类和一些注解。

以最简单的`ClassPathXmlApplicationContex`为例说明IoC容器的初始化过程，首先在构造对象时会调用构造方法，构造方法中有一个叫做**refresh**的方法，它的作用是销毁旧的容器并创建新的容器，也就是初始化的过程。

首先refresh方法会先加一个同步代码块然后执行后续的步骤，第一步是准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符、校验配置文件。然后调用obtainFreshBeanFactory()方法返回一个BeanFactory，这个方法内部具体的过程是，首先关闭旧的BeanFactory然后new一个类型是**DefaultListableBeanFactory**的`BeanFactory`。生成之后会调用**loadBeanDefinition**来加载bean到BeanFactory。

这个`BeanDefinition`代表了一个bean的所有信息，如bean的名称，bean的id，bean的类型，是否为单例，是否懒加载，所有的依赖等信息。因此实现ioc容器的一个最重要的步骤是如何把配置信息转化成BeanDefinition对象。

具体如何和加载BeanDefinition的呢，它会去实例化一个`XmlBeanDefinitionReader`对象，通过它来加载配置信息，首先会根据配置文件的信息比如说文件地址把这个文件读到内存中，因为这个文件是xml格式的，所以会把他转化为一个DOM树，方便后面的操作，后面就是对这棵DOM树进行解析，将其中的标签解析成BeanDefinition并且把它们注册到BeanFactory中，具体就是把BeanDefinition放入一个Map中。这样我们的BeanFactory就得到了所有的`BeanDefinition`，但是此时还没有进行初始化。

Spring 把我们在 xml 配置的 bean 都注册以后，会设置类加载器，然后还会"手动"注册一些特殊的 bean，这些bean有特殊的作用，比如：

最后一步是把那些没有声明为懒加载的bean实例化，并放在一个单例池中。

之后我们使用这个容器一般是通过`getBean`方法来获取一个Bean的实例，这个方法的参数是bean的Name或者是Bean的Class，首先回去单例池中找，如果找到了就返回，否则会去检查当前这个bean所对应的BeanDefinition是否存在，如果存在就回去尝试加载这个`BeanDefinition`的类，然后实例化，最后进行依赖注入，得到bean实例后返回。



##### 3. Spring AOP的原理

首先解释一下什么是AOP，AOP的全称是面向切面编程，在开发的过程中，有很多的代码是与业务无关的，比如说日志的打印等，但是这些代码可能散落在源代码的各个地方，如果以硬编码的形式实现则维护难度比较大，而使用AOP可以预编译或者运行时动态代理的方式对对象的方法进行增强。

Spring的AOP主要使用了两种技术，一是JDK的Proxy类，二是Cglib。Spring AOP作用于IOC容器中的bean，具体的实现是这样的，在从Spring ioc容器中获取bean的过程中，Spring容器提供了一个调用点给用户，具体来说是在创建出bean的实例后，会调用BeanPostProcessor来处理bean，AOP就是在这个过程中对bean的实例进行了动态代理，然后返回的也是代理类。



##### 4. Spring MVC的实现原理

首先，用户从客户端过来的请求会被一个叫做DispatcherServlet的前端控制器拦截，这个DispatcherServlet类继承自Servlet，拦截到请求后，会通过HandlerMapping去查找处理这个请求的uri的handler，因为handler有多种类型，所以会去找到handleAdapter去处理，处理完以后会返回一个ModelAndView，然后前端控制器会把这个ModelAndView交给视图解析器进行解析和渲染，然后把响应发送到客户端。

![](https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/SSM%E5%8E%9F%E7%90%86%E6%80%BB%E7%BB%93/clipboard.png)

1. 用户发送请求至前端控制器DispatcherServlet
2.  DispatcherServlet收到请求后，调用HandlerMapping处理器映射器，请求获取Handle
3. 处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet； 
4. DispatcherServlet 调用 HandlerAdapter处理器适配器
5. HandlerAdapter 经过适配调用 具体处理器(Handler，也叫后端控制器)； 
6. Handler执行完成返回ModelAndView； 
7. HandlerAdapter将Handler执行结果ModelAndView返回给DispatcherServlet； 
8. DispatcherServlet将ModelAndView传给ViewResolver视图解析器进行解析； 
9. ViewResolver解析后返回具体View； 
10. DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中） 、
11. DispatcherServlet响应用户。