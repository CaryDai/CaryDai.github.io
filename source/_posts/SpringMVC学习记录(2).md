---
title: SpringMVC学习记录(2)
date: 2019-06-18 20:31:03
categories: SpringMVC
tags: SpringMVC
---

这篇介绍的是 SpringMVC 的其它常用功能。

## 图片上传处理

1. 首先需要配置 Tomcat 的图片存储目录：在`server.xml`中添加如下代码

   ```xml
   <!-- 配置访问图片。docBase是图片存储的地址，path是访问的地址 -->
   <Context docBase="E:\Pictures" path="/pic" reloadable="true" />
   ```

2. 在`pom.xml`中添加上传文件的 jar 包：

   ```xml
   <!-- 添加上传文件的jar包 -->
   <dependency>
       <groupId>commons-fileupload</groupId>
       <artifactId>commons-fileupload</artifactId>
       <version>1.3.1</version>
   </dependency>
   <dependency>
       <groupId>commons-io</groupId>
       <artifactId>commons-io</artifactId>
       <version>2.4</version>
   </dependency>
   ```

3. 在SpringMVC的配置文件中配置**多媒体处理器**：

   ```xml
   <!-- 配置多媒体处理器 -->
   <!-- 注意：这里id必须填写：multipartResolver -->
   <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
       <!-- 最大上传文件大小，以字节计算 -->
       <property name="maxUploadSize" value="8388608" />
   </bean>
   ```

4. 在上传文件的表单中指定属性：

   ```jsp
   <!-- 上传图片是需要指定属性 enctype="multipart/form-data"。而且要求是post请求 -->
   <form id="itemForm"	action="${pageContext.request.contextPath }/updateItem.action" enctype="multipart/form-data" method="post">
       ...
       <tr>
           <td>商品图片</td>
           <td>
               <c:if test="${item.pic != null}">
                   <img src="/pic/${item.pic}" width=100 height=100/>
                   <br/>
               </c:if>
               <input type="file"  name="pictureFile"/>
           </td>
       </tr>
       ...
   </form>
   ```

5. 编写Controller里提交图片的方法

   ```java
   @RequestMapping(value = "updateItem", method = {RequestMethod.POST, RequestMethod.GET})
   public String updateItem(Item item, MultipartFile pictureFile, Model model) throws Exception {
   
       // 新的图片名字
       String newName = UUID.randomUUID().toString();
       // 图片原来的名字
       String oldName = pictureFile.getOriginalFilename();
       // 图片后缀名
       String sux = oldName.substring(oldName.lastIndexOf("."));
       // 新建本地文件流(图片保存的位置)
       File file = new File("E:\\Pictures\\" + newName + sux);
       // 写入本地磁盘
       pictureFile.transferTo(file);
   
       // 保存图片到数据库
       item.setPic(newName + sux);
   
       itemService.updateItem(item);
       model.addAttribute("item", item);
       model.addAttribute("msg", "修改商品信息成功！");
       return "itemEdit";
   }
   ```


## JSON数据交互

1. 在`pom.xml`中添加 json 依赖的 jar 包：

   ```xml
   <!-- 添加json依赖的jar包 -->
   <dependency>
       <groupId>com.fasterxml.jackson.core</groupId>
       <artifactId>jackson-databind</artifactId>
       <version>2.9.3</version>
   </dependency>
   <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-core -->
   <dependency>
       <groupId>com.fasterxml.jackson.core</groupId>
       <artifactId>jackson-core</artifactId>
       <version>2.9.3</version>
   </dependency>
   <!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-annotations -->
   <dependency>
       <groupId>com.fasterxml.jackson.core</groupId>
       <artifactId>jackson-annotations</artifactId>
       <version>2.9.3</version>
   </dependency>
   ```

2. 在Controller里面编写JSON数据响应的方法：

   ```java
   @RequestMapping("getItem")
   @ResponseBody   // ResponseBody用来响应json串
   public Item getItemJson() {
       Item item = itemService.getItemById(1);
       return item;
   }
   ```

   接收json串使用`RequestBody`：

   ```java
   @RequestMapping("getItem2")
   @ResponseBody   // ResponseBody用来响应json串
   public Item getItemJson(@RequestBody Item itemIn) {     // RequestBody用来接收json串
       System.out.println(itemIn);
       itemIn.setName("披萨");
       return itemIn;
   }
   ```

## RESTful风格的支持

RESTful用于 Web 数据接口的设计。具体内容可以参考这两篇文章：[理解RESTful架构](<http://www.ruanyifeng.com/blog/2011/09/restful.html>)、[RESTful API 最佳实践](<http://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html>)。

```java
// Restful风格：item/1
@RequestMapping("item/{id}")
public String itemQuery(@PathVariable("id") Integer ids, Model model) {
    // 查询商品信息
    Item item = itemService.getItemById(ids);
    // model返回数据模型
    model.addAttribute("item", item);
    return "itemEdit";
}
```

## 拦截器

1. 首先，自定义一个拦截器，需要实现`HandlerInterceptor接口`：

```java
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * 自定义拦截器
 */
public class MyInterceptor implements HandlerInterceptor {
    // 进入方法前被执行（比如：登录拦截、权限校验等等）
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("MyInterceptor.preHandle");
        // true放行，false拦截
        return true;
    }

    // 方法执行之后返回ModelAndView之前被执行（比如：设置页面的共用参数等操作）
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("MyInterceptor.postHandle");
    }

    // 方法执行后被执行（可以处理异常、清理资源、记录日志等操作）
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("MyInterceptor.afterCompletion");
    }
}
```

2. 在配置文件`spring-mvc.xml`中配置拦截器：

```xml
<mvc:interceptors>
    <mvc:interceptor>
        <!-- /**表示拦截所有请求，包括二级以上目录 -->
        <mvc:mapping path="/**"/>
        <bean class="hdu.dqj.interceptor.MyInterceptor" />
    </mvc:interceptor>
</mvc:interceptors>
```

3. 在Controller的`itemList`方法中输出一句话：

```java
@RequestMapping(value = {"itemList", "itemList2"})
public ModelAndView itemList() {
    ModelAndView mav = new ModelAndView();
    List<Item> list = itemService.getItemList();
    // 设置模型数据，用于传递到jsp
    mav.addObject("itemList", list);
    // 设置视图名字，用于响应用户
    mav.setViewName("itemList");
    System.out.println("ItemController.itemList.....");	//输出
    return mav;
}
```

4. 输出顺序为：

```
MyInterceptor.preHandle
ItemController.itemList.....
MyInterceptor.postHandle
MyInterceptor.afterCompletion
```

### 编写登录拦截器

1、有一个登录页面，需要写一个controller访问页面

2、登录页面有一提交表单的动作。需要在controller中处理。

- 判断用户名密码是否正确
- 如果正确 想session中写入用户信息
- 返回登录成功，或者跳转到商品列表

3、拦截器。

- 拦截用户请求，判断用户是否登录
- 如果用户已经登录。放行
- 如果用户未登录，跳转到登录页面。

1. 首先需要编写一个登录的页面：

   ```jsp
   <%@ page contentType="text/html;charset=UTF-8" language="java" %>
   <html>
   <head>
       <title>用户登录</title>
   </head>
   <body>
   <form action="${pageContext.request.contextPath }/user/login.action">
       用户名：<input type="text" name="username"/><br>
       密码：<input type="password" name="password"/><br>
       <input type="submit">
   </form>
   </body>
   </html>
   ```

2. 编写登录的控制器：

   ```java
   /**
    * 用户请求处理器
    */
   @Controller
   @RequestMapping("user")
   public class LoginController {
       @RequestMapping("toLogin")
       public String toLogin() {
           return "login";
       }
   
       @RequestMapping("login")
       public String login(String username, String password, HttpSession session) {
           if (username.equals("admin")) {
               session.setAttribute("username", username);	// 向session中写入用于信息
               return "redirect:/itemList.action";     // 没有/的话默认是当前目录，当前目录是user目录
           }
           return "login";
       }
   }
   ```

3. 编写登录拦截器：

   ```java
   /**
    * 登录拦截器
    */
   public class LoginInterceptor implements HandlerInterceptor {
       // 进入方法前被执行（比如：登录拦截、权限校验等等）
       @Override
       public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
           // 判断用户有没有登录
           Object object = request.getSession().getAttribute("username");
           if (object == null) {
               response.sendRedirect(request.getContextPath() + "/user/toLogin.action");
           }
           // true放行，false拦截
           return true;
       }
   
       // 方法执行之后返回ModelAndView之前被执行（比如：设置页面的共用参数等操作）
       @Override
       public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
       }
   
       // 方法执行后被执行（可以处理异常、清理资源、记录日志等操作）
       @Override
       public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
       }
   }
   ```

4. 接着在配置文件中配置拦截器：

   ```xml
   <mvc:interceptors>
       <mvc:interceptor>
           <!-- /**表示拦截所有请求，包括二级以上目录 -->
           <mvc:mapping path="/**"/>
           <!-- 配置不拦截目录 -->
           <mvc:exclude-mapping path="/user/**" />
           <bean class="hdu.dqj.interceptor.LoginInterceptor" />
       </mvc:interceptor>
   </mvc:interceptors>
   ```

   