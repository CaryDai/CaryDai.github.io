---
title: jsp知识记录
date: 2019-06-05 19:10:30
categories: JSP
tags: jsp
---

## JSP简介

JSP全名为Java Server Pages，中文名叫java服务器页面。它是一种动态页面技术，主要目的是将表示逻辑从 Servlet 中分离出来。

在 idea 中新建一个`.jsp`页面时，在注释信息的下面有一行标签：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
```

这行指令标签定义了页面的依赖属性，contentType 指定了当前JSP页面的MIME类型和字符编码，language 则定义了JSP页面所用的脚本语言，默认是 Java。

## JSTL

JSP标准标签库（JSTL）是一个JSP标签集合，它封装了JSP应用的通用核心功能。JSTL支持通用的、结构化的任务，比如迭代，条件判断，XML文档操作，国际化标签，SQL标签。 除了这些，它还提供了一个框架来使用集成JSTL的自定义标签。（来自菜鸟教程<https://www.runoob.com/jsp/jsp-jstl.html>）

## JSP指令

JSP 中有三种类型的指令：page指令、include指令、taglib指令。

- page指令为容器提供当前页面的使用说明。一个JSP页面可以包含多个page指令。示例如上所示。

- include指令用来包含其他文件。被包含的文件可以是JSP文件、HTML文件或文本文件。包含的文件就好像是该JSP文件的一部分，会被同时编译执行。

- taglib指令引入一个自定义标签集合的定义，包括库路径、自定义标签。
  taglib指令的语法：

  ```jsp
  <!-- uri属性确定标签库的位置，prefix属性指定标签库的前缀 -->
  <%@ taglib uri="uri" prefix="prefixOfTag" %>
  如：
  <!-- 引用核心标签库 -->
  <%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
  <!-- 引用格式化库。JSTL格式化标签用来格式化并输出文本、日期、时间、数字。 -->
  <%@ taglib uri="http://java.sun.com/jsp/jstl/fmt"  prefix="fmt"%>
  ```


## EL表达式

EL表达式允许我们指定一个表达式来表示属性值：`${expr}`。在JSP EL中通用的操作符是`.`和 `{}` 。这两个操作符允许您通过内嵌的JSP对象访问各种各样的JavaBean属性。

```jsp
<%--想要停用对EL表达式的评估的话，需要使用page指令将isELIgnored属性值设为true.这样，EL表达式就会被忽略。若设为false，则容器将会计算EL表达式。--%>
<%@ page isELIgnored="false" %>
```

### JSP EL隐含对象

#### pageContext对象

这个对象代表着页面上下文，也就是当前页面所在的环境。`pageContext`对象是`javax.servlet.jsp.PageContext`类实例，我们可以通过这个对象来访问page、request、session和application作用域下的变量。

```jsp
<form action="${pageContext.request.contextPath }/queryItem.action" method="post">
```

`${pageContext.request.contextPath}`是JSP取得**绝对路径**的方法，等价于`<%=request.getContextPath()%> `。也就是取出部署的应用程序名或者是当前的项目名称。比如我的项目名称是`springmvcmybatis`，在浏览器输入`http://localhost:8082/springmvcmybatis/queryItem.action`，`${pageContext.request.contextPath}`取出来的就是`/springmvcmybatis`，而`/`代表的就是`http://localhost:8082`。

## JSP中的四大作用域

这四个作用域的作用范围，由上到下是一个比一个大。

### page作用域

page直译就是页面的意思，page作用域表示只在当前页面有效。当程序运行跑出了当前的页面，你就无法在其它的页面访问当前页面设置的属性值。

### request作用域

### session作用域

### application作用域

## 参考资料

1. JSP百度百科(<https://baike.baidu.com/item/JSP/141543?fr=aladdin>)
2. JSP菜鸟教程(<https://www.runoob.com/jsp/jsp-directives.html>)
3. 关于${pageContext.request.contextPath}的理解(https://www.cnblogs.com/hy1988/p/5811445.html)
4. JSP中的四大作用域(https://www.jellythink.com/archives/216)

