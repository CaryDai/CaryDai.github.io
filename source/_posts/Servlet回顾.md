---
title: Servlet回顾
date: 2019-09-25 09:19:58
tags: Servlet
categories: Servlet
description: 复习Servlet。
---

## 简介

Java Servlet 是运行在 Web 服务器或应用服务器上的程序，它是作为来自 Web 浏览器或其他 HTTP 客户端的请求和 HTTP 服务器上的数据库或应用程序之间的中间层。

Servlet 在 Web 应用程序中的位置：

![servlet-arch.jpg](http://ww1.sinaimg.cn/large/005Mjp9lly1g7bi59gl6rj308j05jwep.jpg)

Servlet 包：可以使用`javax.servlet`和`javax.servlet.http`包创建。`javax.servlet`包中定义了所有的 Servlet 类都必须实现或扩展的的通用接口和类；`javax.servlet.http`包中定义了采用 HTTP 通信协议的 HttpServlet 类，它在原有 Servlet 接口的基础上添加了一些与 Http 协议相关的处理方法，在编写 Servlet 时一般继承这个接口。

侠义的 Servlet 是指`javax.servlet`包中定义的一个接口，广义的 Servlet 是指任何实现了这个 Servlet 接口的类。

## Servlet 生命周期

Servlet 遵循的过程：

- Servlet 通过调用 **init ()** 方法进行初始化。
- Servlet 调用 **service()** 方法来处理客户端的请求。（每次服务器接收到一个 Servlet 请求时，服务器会产生一个新的线程并调用服务。service() 方法检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut，doDelete 等方法。）
- Servlet 通过调用 **destroy()** 方法终止（结束）。
- 最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。

## 响应过程

Servlet 容器：

![Servlet容器.jpg](http://ww1.sinaimg.cn/large/005Mjp9lly1g7bjlxk19rj30ev05v0sq.jpg)

HttpServlet 容器响应 Web 客户请求流程如下：

1. Web 客户向 Servlet 容器发出 Http 请求；
2. Servlet 容器解析 Web 客户的 Http 请求；
3. Servlet 容器创建一个 HttpRequest 对象，在这个对象中封装 Http 请求信息；
4. Servlet 容器创建一个 HttpResponse 对象；
5. Servlet 容器调用 HttpServlet 的 service 方法，把 HttpRequest 和 HttpResponse 对象作为 service 方法的参数传给 HttpServlet 对象；
6. HttpServlet 调用 HttpRequest 的有关方法，获取 HTTP 请求信息；
7. HttpServlet 调用 HttpResponse 的有关方法，生成响应数据；
8. Servlet 容器把 HttpServlet 的响应结果传给 Web 客户。

创建 HttpServlet 的步骤——“四部曲”：

1. 扩展 HttpServlet 抽象类；
2. 覆盖 HttpServlet 的部分方法，如覆盖 doGet() 或 doPost() 方法；
3. 获取 HTTP 请求信息。通过 HttpServletRequest 对象来检索 HTML 表单所提交的数据或 URL 上的查询字符串；
4. 生成 HTTP 响应结果。通过 HttpServletResponse 对象生成响应结果。

## 参考资料

1. [Servlet菜鸟教程](http://www.runoob.com/servlet/servlet-first-example.html)

2. [HttpServlet详解](https://www.cnblogs.com/panjun-Donet/archive/2010/02/22/1671290.html)