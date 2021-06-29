---
title: SpringMVC学习记录(1)
date: 2019-06-05 19:40:36
categories: SpringMVC
tags: SpringMVC
---

## SpringMVC简介

SpringMVC 是一个 web 层框架，用来处理浏览器请求的解析和视图的展示。

IDEA 中创建 SpringMVC 项目可以参考这篇博客：<https://www.cnblogs.com/chenlinghong/p/8339555.html>

先给出SpringMVC的总体架构图：

![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3qkej9eaqj311y0lcwj0.jpg)

### 前端控制器DispacherServlet

主要用作职责调度工作，需要在`web.xml`中配置。

### ModelAndView

ModelAndView = Model + View。
View 由`setViewName()`方法来决定，决定让`视图解析器ViewResolver`去哪里找View文件，并指明找到的 jsp 文件名。
Model 由`addObject()`方法来决定，用于设定模型数据，传递到 jsp。
“**用人话来解释ModelAndView的功能就是，View负责渲染Model，让你找到代表View的jsp，用这个jsp去渲染Model中的数据。**”

### 视图解析器ViewResolver

从图中可以看到视图解析器用于解析视图，并返回View对象。

## 第一个 Demo

1. 首先创建一个`hello.jsp`页面：

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>输出提示</title>
</head>
<body>
<%-- EL表达式 --%>
${ msg }
</body>
</html>
```

2. 然后写一个 Controller：

```java
@Controller
public class HelloController {
    @RequestMapping("hello")    // 绑定请求地址
    public ModelAndView hello() {
        System.out.println("Hello Springmvc...");
        ModelAndView mav = new ModelAndView();
        // 设置模型数据，用于传递到jsp
        mav.addObject("msg", "hello springmvc....");
        // 设置视图名字，用于响应用户
        mav.setViewName("/WEB-INF/jsp/hello.jsp");
        return mav;
    }
}
```

3. 创建与配置 SpringMVC 的配置文件：dispatcher-servlet.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">

    <!-- 配置controller扫描包 -->
    <context:component-scan base-package="hdu.dqj.springmvc.controller" />
</beans>
```

4. 在 web.xml 中配置前端控制器

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- 加载springmvc核心配置文件 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <!-- 设置拦截地址 -->
        <url-pattern>*.action</url-pattern>
    </servlet-mapping>
</web-app>
```

5. 启动项目测试

![](http://ww1.sinaimg.cn/large/005Mjp9lgy1g3qje4s71ij30cs03nglh.jpg)

### 几点说明

1. `dispatcher-servlet.xml` 中的 `<context:component-scan>`元素是启动“包扫描”功能。`base-package`的值是包的路径。这样就会扫描`hdu.dqj.springmvc.controller`这个包及其子包下的所有类。
2. `HelloController`类上面有一个`@Controller`注解，用于指明该类就是一个控制器类。SpringMVC 会依据controller中处理方法的返回值，进行解析，解析之后提供一个视图，作为响应。
3. 代码的执行流程是`5-> 4-> 3-> 2-> 1`。

## 第二个Demo

目的：完成商品列表的加载。

1. 创建 Item 的 pojo

   ```java
   import java.util.Date;
   /**
    * 商品数据模型
    */
   public class Item {
       // 商品id
       private int id;
       // 商品名称
       private String name;
       // 商品价格
       private double price;
       // 商品创建时间
       private Date createtime;
       // 商品描述
       private String detail;
       // 带参数的构造器和set/get方法
   }
   ```

2. 编写ItemController

   ```java
   @Controller //负责注册一个bean 到spring 上下文中
   public class ItemController {
       @RequestMapping("itemList")     //注解为控制器指定可以处理哪些 URL 请求
       public ModelAndView itemList() {
           ModelAndView mav = new ModelAndView();
           // 模拟查询商品列表
           // asList将数组转为List，但不能处理基本类型数据。
           List<Item> list = Arrays.asList(new Item(1, "冰箱", 3000, new Date(), "海尔冰箱"),
                   new Item(2, "冰箱2", 3000, new Date(), "海尔冰箱2"),
                   new Item(3, "冰箱3", 3600, new Date(), "海尔冰箱3"),
                   new Item(4, "冰箱4", 3800, new Date(), "海尔冰箱4"));
           // 设置模型数据，用于传递到jsp
           mav.addObject("itemList", list);
   //        mav.setViewName("/WEB-INF/jsp/itemList.jsp");
           mav.setViewName("itemList");	// 在SpringMVC配置文件中配置了视图解析器
           return mav;
       }
   }
   ```

3. 编写itemList.jsp

   ```jsp
   <%@ page language="java" contentType="text/html; charset=UTF-8"
       pageEncoding="UTF-8"%>
   <%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
   <%@ taglib uri="http://java.sun.com/jsp/jstl/fmt"  prefix="fmt"%>
   <!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
   <html>
   <head>
   <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
   <title>查询商品列表</title>
   </head>
   <body> 
   <form action="${pageContext.request.contextPath }/queryItem.action" method="post">
   查询条件：
   <table width="100%" border=1>
   <tr>
   <td><input type="submit" value="查询"/></td>
   </tr>
   </table>
   商品列表：
   <table width="100%" border=1>
       <tr>
           <td>商品名称</td>
           <td>商品价格</td>
           <td>生产日期</td>
           <td>商品描述</td>
           <td>操作</td>
       </tr>
       <c:forEach items="${itemList }" var="item">
       <tr>
           <td>${item.name }</td>
           <td>${item.price }</td>
           <td><fmt:formatDate value="${item.createtime}" pattern="yyyy-MM-dd HH:mm:ss"/></td>
           <td>${item.detail }</td>
           <td><a href="${pageContext.request.contextPath }/itemEdit.action?id=${item.id}">修改               </a></td>
       </tr>
       </c:forEach>
   </table>
   </form>
   </body>
   </html>
   ```
   
4. 修改dispacher-servlet.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
          xmlns:context="http://www.springframework.org/schema/context"
          xmlns:mvc="http://www.springframework.org/schema/mvc"
          xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
           http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">
   
       <!-- 配置controller扫描包 -->
       <context:component-scan base-package="hdu.dqj.springmvc.controller" />
   
       <!-- 配置注解驱动，相当于同时配置了最新的处理器映射器和处理器适配器，对json数据的响应提供支持 -->
       <mvc:annotation-driven />
   
       <!--指定视图解析器-->
       <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
           <!-- 视图的路径 -->
           <property name="prefix" value="/WEB-INF/jsp/" />
           <!-- 视图名称后缀  -->
           <property name="suffix" value=".jsp" />
       </bean>
   </beans>
   ```

5. 启动运行(<http://localhost:8082/day01/itemList.action>)
   ![](http://ww1.sinaimg.cn/large/005Mjp9lgy1g3qm3kfltdj311o060aa9.jpg)

## 参数绑定

### 默认支持的参数类型

```java
/**
     * 根据id查询商品信息，跳转修改商品页面。
     * 演示默认支持的参数传递
     * Model/ModelMap：返回数据模型
     * @param request
     * @param response
     * @param session
     * @return
     */
@RequestMapping("itemEdit")
public String itemEdit(Model model, HttpServletRequest request, HttpServletResponse response, HttpSession session) {
    String idStr = request.getParameter("id");  // 通过request取得从itemList页面传递过来的参数

    System.out.println("response: " + response);
    System.out.println("session: " + session);

    // 查询商品信息
    Item item = itemService.getItemById(new Integer(idStr));
    // model返回数据模型，将查询到的商品信息传递给 model attribute，即传给itemList.jsp中的item
    model.addAttribute("item", item);
    return "itemEdit";
}
```

`Model`与`ModelMap`类似，都只是用来传输数据，且由`SpringMVC`自动创建。但`ModelAndView`需要由我们手工创建，不过它可以指明要请求的`jsp`文件。

### 简单参数传递

```java
/**
 * 简单参数的传递
 * @RequestParam用法：入参名字与方法名参数名不一致时使用
 * value:传入的参数名，required：是否必填,defaultValue:默认值
 * @param model
 * @param ids
 * @return
 */
@RequestMapping("itemEdit")
public String itemEdit2(Model model, @RequestParam(value = "id", required = true, defaultValue = "1") Integer ids) {
    // 查询商品信息
    Item item = itemService.getItemById(ids);
    // model返回数据模型，将查询到的商品信息传递给 model attribute，即传给itemList.jsp中的item
    model.addAttribute("item", item);
    return "itemEdit";
}
```

### 绑定pojo对象

```java
/**
 * 修改商品
 * 演示pojo参数绑定
 * 要点：表单提交的name属性必需与pojo的属性名称一致。
 * @param item
 * @return
 */
@RequestMapping("updateItem")
public String updateItem(Item item, Model model) {
    // 更新商品
    itemService.updateItem(item);
    model.addAttribute("item", item);
    model.addAttribute("msg", "修改商品信息成功！");
    // 返回修改商品页面
    return "itemEdit";
}
```

### 数组类型的参数绑定

```java
/**
 * springmvc数组参数的绑定
 * @param vo
 * @param ids
 * @param model
 * @return
 */
@RequestMapping("queryItem")
public String queryItem(QueryVo vo, Integer[] ids, Model model) {
    if (vo.getItem() != null) {
        System.out.println(vo.getItem());
    }
    if (ids != null && ids.length > 0) {
        for (Integer id : ids) {
            System.out.println(id);
        }
    }
    // 模拟搜索商品
    List<Item> list = itemService.getItemList();
    model.addAttribute("itemList", list);
    return "itemList";
}
```

### List类型的参数绑定

```java
/**
 * List类型的绑定
 * @param vo
 * @param model
 * @return
 */
@RequestMapping("queryItem")
public String queryItem(QueryVo vo, Model model) {
    if (vo.getItems() != null && vo.getItems().size() > 0) {
        for (Item item : vo.getItems()) {
            System.out.println(item);
        }
    }
    // 模拟搜索商品
    List<Item> list = itemService.getItemList();
    model.addAttribute("itemList", list);
    return "itemList";
}
```

items 是QueryVo类 里面的参数，jsp中需要获取 items 的下标，此时需要用到`c:forEach`标签的`varStatus`属性，这个属性描述了循环的状态信息。

```jsp
<c:forEach items="${itemList }" var="item" varStatus="status">
    <tr>
        <td>
            <%--隐藏域是用来收集或发送信息的不可见元素，对于网页的访问者来说，隐藏域是看不见的。
                        当表单被提交时，隐藏域就会将信息用你设置时定义的名称和值发送到服务器上。--%>
            <input type="hidden" name="items[${status.index}].id" value="${item.id}">
            <input type="text" name="items[${status.index}].name" value="${item.name}"></td>
        <td><input type="text" name="items[${status.index}].price" value="${item.price}"></td>
        <td>
            <input type="text" name="items[${status.index}].createtime"
                   value='<fmt:formatDate value="${item.createtime}" pattern="yyyy-MM-dd HH:mm:ss"/>'>
        </td>
        <td><input type="text" name="items[${status.index}].detail" value="${item.detail}"></td>
        <td><a href="${pageContext.request.contextPath }/itemEdit.action?id=${item.id}">修改</a></td>

    </tr>
</c:forEach>
```

SpringMVC并不能识别日期的格式，此时需要自定义一个日期转换器：

```java
import org.springframework.core.convert.converter.Converter;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
/**
 * SpringMVC不认识日期的格式，所以自定义日期转换器
 * S：source 要转换的源类型
 * T：目标，要转换成的数据类型
 */
public class DateConvert implements Converter<String, Date> {
    @Override
    public Date convert(String source) {
        Date result = null;
        try {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            result = sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return result;
    }
}
```

接着需要在`spring-mvc.xml`中配置这个转换器：

```xml
<!-- 定义转换器 -->
<bean id="MyConvert" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="converters">
        <set>
        	<bean class="hdu.dqj.utils.DateConvert" />
        </set>
    </property>
</bean>
```

随后在`mvc:annotation-driven`中使用自定义转换器

```xml
<!-- 使用自定义转换器 -->
<mvc:annotation-driven conversion-service="MyConvert" />
```

## @RequestMapping注解的几点说明

1. @RequestMapping 可以绑定多个连接

   ```java
   @RequestMapping(value = {"itemList", "itemList2"})
   public ModelAndView itemList() {
       ModelAndView mav = new ModelAndView();
       List<Item> list = itemService.getItemList();
       // 设置模型数据，用于传递到jsp
       mav.addObject("itemList", list);
       // 设置视图名字，用于响应用户
       mav.setViewName("itemList");
       System.out.println("ItemController.itemList.....");
       return mav;
   }
   ```

2. @RequestMapping 也可以加在类的上面，作为分目录管理

3. @RequestMapping 还可以限定请求方法：

   ```java
   // 配置了method表示支持的请求类型，如果没写method，则默认支持所有类型的请求。
   @RequestMapping(value = "updateItem", method = {RequestMethod.POST, RequestMethod.GET})
   ```

   解决`Get请求`的乱码问题：在Tomcat的`server.xml`中添加`URIEncoding="UTF-8"`：

   ```xml
   <Connector port="8082" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" 			URIEncoding="UTF-8"/>
   ```

   解决`Post请求`的乱码问题：在项目的`web.xml`文件中添加以下代码：

   ```xml
   <!-- 解决post乱码问题 -->
   <filter>
       <filter-name>encoding</filter-name>
       <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
       <!-- 设置编码是UTF8 -->
       <init-param>
           <param-name>encoding</param-name>
           <param-value>UTF-8</param-value>
       </init-param>
   </filter>
   <filter-mapping>
       <filter-name>encoding</filter-name>
       <!-- 拦截所有请求 -->
       <url-pattern>/*</url-pattern>
   </filter-mapping>
   ```

## Controller方法的返回值

### 返回ModelAndView

参考之前的内容。

### 返回`void`

1. 通过 request 返回

   ```java
   @RequestMapping("queryVoid")
   public void queryVoid(HttpServletRequest request, HttpServletResponse response) throws Exception {
       // request响应用户请求
       request.setAttribute("msg", "这个是request响应的消息");
       request.getRequestDispatcher("/WEB-INF/jsp/msg.jsp").forward(request, response);
   }
   ```

2. 通过 response 返回

   ```java
   @RequestMapping("queryVoid")
   public void queryVoid(HttpServletRequest request, HttpServletResponse response) throws Exception {
       // response响应用户请求（重定向）
       //        response.sendRedirect("/itemList.action");
       
       // 不经过视图解析器时，需要设置编码；经过视图解析器的话，SpringMVC会帮我们转码
       response.setContentType("text/html;charset=utf-8");
       PrintWriter printWriter = response.getWriter();
       printWriter.println("这个是response打印的消息");
   }
   ```

### 返回String

参考之前写的。

还可以通过请求转发和重定向的方式返回（相当于`request`的请求转发和`response`的请求重定向）：

```java
return "forward:itemList.action";   // forward是请求转发
return "redirect:itemList.action";
```

## SpringMVC中异常的处理

可以做一个全局异常处理器，处理所有没有处理过的运行时异常，用于更好地提示用户。
首先定义一个自定义异常类`MyException.java`：

```java
/**
 * 自定义异常
 */
public class MyException extends Exception {
    private String msg;

    public MyException() {
    }

    public MyException(String msg) {
        this.msg = msg;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

接着定义全局异常处理器：

```java
/**
 * 全局异常处理器
 */
public class CustomerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) {
        String result = "系统发生异常，请联系管理员。";
        // 自定义异常处理
        // instanceof 关键字用来在运行时指出对象是否是特定类的一个实例。
        if (e instanceof MyException) {
            result = ((MyException)e).getMsg();
        }
        ModelAndView mav = new ModelAndView();
        mav.addObject("msg", result);
        mav.setViewName("msg");
        return mav;
    }
}
```

在`spring-mvc.xml`中通过一个`bean`标签配置全局异常处理器：

```java
<!-- 配置全局异常处理器 -->
<bean class="hdu.dqj.exception.CustomerExceptionResolver"/>
```

此时就可以在Controller方法内自定义异常信息：

```java
// 假设这里是根据id查询商品信息，搜索不到商品
if (true) {
    throw new MyException("你查找的商品不存在，请确认信息！");
}
```

当实际使用中抛出异常时，将会交给全局异常处理器处理。

## 参考资料

1. [SpringMVC中ModelAndView对象与“视图解析器”](<https://yq.aliyun.com/articles/229499>)
2. [**Spring中Model,ModelMap以及ModelAndView之间的区别**](<https://blog.csdn.net/zhangxing52077/article/details/75193948>)