---
title: Tomcat Servlet 工作原理
date: 2019-04-15 01:03:40
updated: 2019-04-15 01:03:40
tags: JavaWeb
categories: Java
---
> 本文译自：[An Introduction to Tomcat Servlet Interactions](https://www.mulesoft.com/tcat/tomcat-servlet)
> 译者：喝酒不骑马
> 邮箱：myz9412@163.com

---

# Tomcat Servlet 功能介绍
Apache Tomcat 因其灵活的配置和交互能力其实是作为通用的 web 应用服务运行在多种环境上的，但主要还是作为一个 Java Servlet 容器。

Tomcat 利用它对 Java Servlet 和 JSP API 的实现，可以从客户端接收请求，通过动态编译容器管理的 Java class 文件来处理相关应用程序上下文中指定的请求，并将结果返还给客户端。这种动态生成的特点是快速、多线程，同时还不依赖于特有的平台去处理请求。

除此之外，Java Servlet 的设计规范兼容大部分其它主流的 Java Web 技术，因此托管在 Tomcat 服务上的 servlet 可以使用所有 Tomcat 所支持的资源。通过 Tomcat 嵌套分层式 XML 配置文件可以对资源的访问进行详细地控制，同时还有松耦合、易发布、逻辑性强、可读性好的特点。

本文，我们将讨论 Apache Tomcat 是如何快速地将这些动态内容分发给客户端的。

## Tomcat 是如何处理 Servlets 的？
Servlet 规范中的一个关键要求是，对于整个数据处理过程，Servlet 只负责处理指定的部分。例如，servlet 中的代码不会监听某个特定端口上的请求，也不会直接和客户端打交道，也不负责管理资源的访问权限。而这些琐事都是交给 Tomcat 的 sevlet 容器掌管的。

这么做可以使 servlet 在各种环境中被复用，也可以让其他人并行开发某些模块。也就是说，只要不是一些重大的改动，比如想优化效率，那么不需要改动 servlet 代码，只需要重构连接器就好了。

## Selvet 的生命周期
作为一个被 Tomcat 管理的组件，servlet 有它的生命周期。通常情况下，当有请求时，容器会加载 servlet 的 class 文件，这标志着 servlet 生命周期的开始。当容器通过调用 destroy 方法来关闭 servlet 时，servlet 的生命周期结束。servlet 在这两个节点中的活动被视作是它的生命周期。

一个典型的 servlet 在 Tomcat 上运行时的生命周期如下所示：

1. Tomcat 通过它其中个一个连接器收到了一个客户端的请求。
2. Tomcat 将这个请求映射到一个合适的引擎（Sevlet）来处理。这些引擎包含在其它的一些信息中（例如主机、服务器等信息），而这些信息缩小了 Tomcat 搜索正确引擎的范围。
3. 当请求被映射到一个对应的 servlet，Tomcat 会检查这个 servlet 的 class 文件是否已经被加载。如果没有的话，Tomcat 会将 servlet 编译成可以被 JVM 所执行的 Java 字节码，并创建一个 servlet 实例。
4. Tomcat 通过调用 servlet 的 init 方法来对它进行初始化。Servlet 中包含着读取 Tomcat 配置文件并能相应地执行操作的代码，也包含声明哪些资源可能会被使用的代码，这使 Tomcat 能够以有序的、可管理的方式来创建 servlet。
5. 当 servlet 被初始化后，Tomcat 可以调用 servlet 中的业务相关的方法来处理这个请求，并作为一个响应（response）返回。
6. 在整个 servlet 的生命周期里，Tomcat 和 servlet 之间可以使用监听器来进行通讯，这些监听器监视着 servlet 的变化。Tomcat 可以用多种方式来检索和存储这些变化，并允许其它的 servlet 能够获取这些信息，从而允许那些给定了上下文的各个组件在一个或多个用户会话间维护或访问这些信息。例如电商应用可以在购物车中记录用户添加的商品，而在结账环节中使用这些数据
7. Tomcat 通过调用 servlet 的 destroy 方法来“委婉地”删除 servlet。这个操作可以由更改被监听的状态来触发，或者通过发送给 Tomcat 的用于取消 servlet 部署的外部命令触发，亦或是通过关闭服务器来触发。

## 组件的协作
通过使用 servlet 以及访问由 HTML 和 JSP（HTML 和 Java 的混合体）组成的资源，并使用本地标签库或自定义标签调用 servlet 方法的方式，Tomcat 可以给用户提供一个动态的、安全的、可持久化的 web 应用。

举个例子，用户可以访问一个页面，这个页面是由客户端通过 AJAX、CSS 以及 HTML 与 DOM 交互的方式来处理动态用户对象的。而用户信息则通过与 servlet 方法交互的 JSP 标签来从数据库中获取。这使页面的展示功能和业务逻辑完全分离，从而提高了安全性和设计的灵活性。

## 延展阅读
在本问的基础上，我们提供了几个深入讲解 Tomcat 如何使用 servlet 的文章：

- [Tomcat Deploy](https://www.mulesoft.com/tcat/tomcat-deploy-procedures)——简单介绍关于Tomcat的部署以及如何配置 servlet。
- [Tomcat JSP](https://www.mulesoft.com/tomcat-jsp)——深入理解 JSP（一种使用 servlet 规范元素的 Java 技术），还包括 JSP 架构的概览以及最佳实践。
- [Tomcat Context](https://www.mulesoft.com/tcat/tomcat-context)——一篇通俗易懂的，对 Tomcat 基于容器的管理模型的全面介绍文章，这对于开始使用 servlet 的人来说很有帮助。