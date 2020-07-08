# 1、Spring 概述

## 1.1、Spring 简介

Spring是Java EE编程领域的一个轻量级开源框架，该框架由一个叫Rod Johnson的程序员在 2002 年最早提出并随后创建，是为了解决企业级编程开发中的复杂性，实现敏捷开发的应用型框架 。 Spring是一个开源容器框架，它集成各类型的工具，通过核心的Bean factory实现了底层的类的实例化和生命周期的管理。在整个框架中，各类型的功能被抽象成一个个的 Bean，这样就可以实现各种功能的管理，包括动态加载和切面编程。

官方网址：http://spring.io/

## 1.2、Spring 发展历程

- 1997年 IBM 提出了 EJB 的思想；1998年，SUN 制定了开发标准规范 EJB1.0；1999年，EJB1.1 发布；2001年，EJB2.0 发布；2003年，EJB2.1发布；2006年，EJB3.0发布；
- Rod Johnson（Spring 之父）
  - Expert One-to-One J2EE Design and Development（2002）阐述了J2EE使用EJB开发设计的优点以及解决方案
  - Expert One-to-One J2EE Development without EJB（2004）阐述了J2EE开发不使用EJB的解决方案（Spring 雏形）
- 2007年9月发布了Spring 的最新版本 Spring 5.0 GA 通用版本

## 1.3、Spring 的优势

- **方便解耦，简化开发**

  通过Spring提供的IoC容器，我们可以将对象之间的依赖关系交由Spring进行控制，避免硬编码所造成的过度程序耦合。有了Spring，用户不必再为单实例模式类、属性文件解析等这些很底层的需求编写代码，可以更专注于上层的应用。

- **AOP编程支持**

  通过Spring提供的[AOP](https://baike.baidu.com/item/AOP)功能，方便进行面向切面的编程，许多不容易用传统OOP实现的功能可以通过AOP轻松应付

- **声明式事务支持**

  在Spring中，我们可以从单调烦闷的事务管理代码中解脱出来，通过声明式方式灵活地进行事务的管理，提高开发效率和质量

- **方便程序测试**

  可以用非容器依赖的编程方式进行几乎所有的测试工作，在Spring里，测试不再是昂贵的操作，而是随手可做的事情。例如：Spring对Junit4支持，可以通过注解方便的测试Spring程序。

- **方便集成各种优秀的框架**

  Spring不排斥各种优秀的开源框架，相反，Spring可以降低各种框架的使用难度，Spring提供了对各种优秀框架（如Struts,Hibernate、Hessian、Quartz）等的直接支持

- **降低J2EE API 使用难度**

  Spring对很多难用的Java EE API（如JDBC，JavaMail，远程调用等）提供了一个薄薄的封装层，通过Spring的简易封装，这些Java EE API的使用难度大为降低

- **源码是经典的 J2EE学习典范**

  Spring的源码设计精妙、结构清晰、匠心独运，处处体现着大师对[Java设计模式](https://baike.baidu.com/item/Java设计模式)灵活运用以及对Java技术的高深造诣。Spring框架源码无疑是Java技术的最佳实践范例。如果想在短时间内迅速提高自己的Java技术水平和应用开发水平，学习和研究Spring源码将会使你收到意想不到的效果。

## 1.4、Spring 的核心结构

Spring 框架是一个分层架构，由 7 个定义良好的模块组成。Spring 模块建立在核心容器之上，核心容器定义了创建、配置和管理 bean 的方式，如图所示：

![Spring模块组成](01-Spring%20%E6%A6%82%E8%BF%B0/spring-moudles.png)

1. **spring core**：提供了spring框架的核心功能。BeanFactory是spring核心容器的只要组件。它通过控制反转将应用程序的配置和依赖性规范与实际的应用程序代码分开，这是整个spring的基础。
2. **spring context**：通过配置文件，向spring框架提供上下文信息。它构建在BeanFactory之上，另外增加了国际化，资源访问等功能
3. **spring aop**：spring提供了面向方面编程的功能，因为spring的核心是基于控制反转的，所以可以很容易地使spring的依赖注入为aop提供支持
4. **spring dao**：提供了一个简单而又有效的jdbc应用，使用它的dao就足以应付开发人员的日常应用了
5. **spring orm**：spring除了有自己的jdbc应用之外，还提供了对其他一些orm框架的支持。基于spring的良好设计，这些开源框架都可以和spring进行良好的结合
6. **spring web**：提供了简化的除了多部分请求以及将请求参数绑定到域对象的任务
7. **spring mvc**：spring提供了MVC2模式的实现，使用起来非常方便，但它不强迫开发人员使用。如果开发人员对其他的MVC框架比较熟悉，仍然可以使用它们。spring对此提供了很好的支持

## 1.5、Spring 框架版本

![Spring 框架版本](01-Spring%20%E6%A6%82%E8%BF%B0/spring-version.png)

