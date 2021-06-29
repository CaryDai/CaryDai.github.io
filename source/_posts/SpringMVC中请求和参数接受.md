---
title: SpringMVC中请求和参数接收
date: 2019-09-25 14:21:43
tags: SpringMVC
categories: SpringMVC
description: 介绍几种网络请求的方式和SpringMVC中参数接收的方式。
---

## HTTP请求方式

我觉得首先需要从 HTTP 协议说起。

HTTP协议（HyperText Transfer Protocol，超文本传输协议）是因特网上应用最为广泛的一种网络传输协议，所有的WWW文件都必须遵守这个标准。HTTP基于TCP/IP通信协议来传递数据（HTML 文件, 图片文件, 查询结果等）。

而所谓的请求方式其实指的就是 HTTP 协议的请求方法，目前为止一共有下面9种，常用的我用红框标明了：

![HTTP的四种请求方法.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7am8p7un6j30o80dwgme.jpg)

简单地说，GET就是获取资源，POST就是创建资源，PUT就是更新资源，DELETE就是删除资源。

关于==GET请求和POST请求的区别==，网上说了很多，总结有以下几点：

- 使用上的区别：
  1. GET使用URL或Cookie传参，而POST将数据放在BODY中。
  2. GET方式提交的数据有长度限制，则POST的数据则可以非常大。（这里长度的限制是对于浏览器和服务器而言）
  3. POST比GET安全，因为数据在地址栏上不可见。（通过GET提交数据，用户名和密码将明文出现在URL上，因为登录页面有可能被浏览器缓存，其他人查看浏览器的历史纪录，那么别人就可以拿到你的账号和密码了）
- 终极区别：
  1. GET请求是幂等性的（幂等性是指一次和多次请求某一个资源应该具有同样的副作用。简单来说意味着对同一URL的多个请求应该返回同样的结果。），POST请求不是。
  2. GET产生一个TCP数据包；POST产生两个TCP数据包。

## SpringMVC中参数接受的方式

### GET请求参数获取

约定成俗，GET请求参数一般都是直接挂在请求的 url 上。

1. SpringMVC 本质上就是一个 Servlet 容器，所以第一种方式可以使用`HttpServletRequest`对象来获取参数。

   ```java
   package com.dqj.get.controller;
   
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   import javax.servlet.http.HttpServletRequest;
   
   /**
    * @Author dqj
    * @Date 2019/9/25 10:42
    * @Version 1.0
    * @Description
    */
   @RestController
   @RequestMapping(value = "web")
   public class ControllerTest {
   
       @RequestMapping(value = "req1")
       public String req1(HttpServletRequest request) {
           String user = request.getParameter("user");
           String password = request.getParameter("password");
           return "user:" + user + ", password:" + password;
       }
   }
   ```

   根据接口在浏览器输入栏输入以下内容：

   ```bash
   http://localhost:8080/web/req1?user=dai&password=123
   # 输出：user:dai, password:123
   
   http://localhost:8080/web/req1?user=dai
   # 输出：user:dai, password:null
   
   http://localhost:8080/web/req1?password=123
   # 输出：user:null, password:123
   ```

2. 根据方法参数获取参数
   还是在同一个类下，编写第二个方法：

   ```java
   @RequestMapping(value = "req2")
   public String req2(String username,String password) {
       return "user: " + username + ", password:" + password;
   }
   ```

   根据接口在浏览器输入栏输入以下内容：

   ```bash
   http://localhost:8080/web/req2?username=dai&password=123
   # 输出：user: dai, password:123
   
   http://localhost:8080/web/req2?user=dai&password=123
   # 输出：user: null, password:123
   
   http://localhost:8080/web/req2?username=dai&Password=123
   # 输出：user: dai, password:null
   
   http://localhost:8080/web/req2?password=123&username=dai
   # 输出：user: dai, password:123
   ```

   注意：

   - 上面这种使用方式，相当于直接将 url 参数映射到了 Controller 方法的参数上了；
   - 方法参数名必须和 url 参数名完全一致（区分大小写）
   - 顺序无关

3. RequestParam 注解方式获取参数
   还是在同一个类下，编写第三个方法：

   ```java
   @RequestMapping(value = "req3")
   public String req3(@RequestParam("user") String username,
                      @RequestParam("pwd") String password) {
       return "user:" + username + ", password:" +password;
   }
   ```

   根据接口在浏览器输入栏输入以下内容：

   ```bash
   http://localhost:8080/web/req3?user=dai&pwd=123
   # 输出：user:dai, password:123
   
   http://localhost:8080/web/req3?username=dai&pwd=123
   # 报错：There was an unexpected error (type=Bad Request, status=400).Required String parameter 'user' is not present
   
   http://localhost:8080/web/req3?user=dai&username=dqj&pwd=123
   # 输出：user:dai, password:123
   ```

   注意：

   - 不指定`@RequestParam`注解的 name 或 value 属性时，等同于第二种使用姿势；

   - url 中的参数名和`@RequestParam`注解中的参数名需保持一致；

   - 参数类型需要保持一致（第二种方法的参数类型也需要保持一致）；

   - 可以通过设置`@RequestParam`注解的`required`属性为 false 使得某个 url 参数可选：

     ```java
     @RequestParam(value = "pwd", required = false) String password
     ```

     此时在浏览器中输入：http://localhost:8080/web/req3?user=dai，并不会报错，返回：`user:dai, password:null`。

4. Bean方式获取参数
   对于请求参数比较复杂的情况，可以使用这种方式，管理起来方便。

   ```java
   @Data
   private static class User {
       private String user;
       private String password;
   }
   
   @RequestMapping(value = "req4")
   public String req4(User user) {
       return "user:" + user;
   }
   ```

   根据接口在浏览器输入栏输入以下内容：

   ```bash
   http://localhost:8080/web/req4?user=dai&password=123
   # 输出：user:ControllerTest.User(user=dai, password=123)
   
   http://localhost:8080/web/req4?user=dai
   # 输出：user:ControllerTest.User(user=dai, password=null)
   ```

   注意：

   - 定义一个 bean，内部属性和请求参数对应；
   - 允许参数不存在的情况，会使用 null 代替（所以，尽量不要使用非封装基本类型，否则参数不传时，会抛异常）。

5. Path 参数
   所谓的 Path 参数指的就是 url 路径中的参数，比如博客园中的一篇文章的地址为：`https://www.cnblogs.com/logsharing/p/8448446.html`，这里的`8448446`这个参数用来区分文章。

   ```java
   @RequestMapping(value = "req5/{user}/info")
   public String req5(@PathVariable(name = "user") String username) {
       return "user:" + username;
   }
   ```

   根据接口在浏览器输入栏输入以下内容：

   ```bash
   http://localhost:8080/web/req5/dai/info
   # 输出：user:dai
   
   http://localhost:8080/web/req5/dai/info?user=kobe
   # 输出：user:dai
   
   http://localhost:8080/web/req5/dai?user=kobe
   # 报错：There was an unexpected error (type=Not Found, status=404).No message available
   ```

   注意：

   - path 参数的使用，需要确保参数存在且类型匹配；

### POST请求参数获取

POST请求参数，更多的是看提交表单参数是否可以获取到，以及如何获取，主要的手段依然是上面几种方式。

1. HttpServletRequest 方式获取参数
   在 Postman 中选择 POST 连接方法，输入连接路径。在 Headers 中设置 Key 为`Content-Type`，Value 为 `application/x-www-form-urlencoded`：
   ![Postman1.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7bri3ovz4j30td06waae.jpg)
   接着在 Body 中选择 `x-www-form-urlencoded`这个数据格式，写入 Key 和 Value 值：
   ![Postman2.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7bribxw3mj30tb05jjri.jpg)
   这样发送请求就能返回数据了。

   ```bash
   # 上图的情况输出：user:dai, password:123
   ```

   说明：

   - 对于 HttpServletReuqest 方式获取参数时，get 和 post 没什么区别；
   - 若 url 参数和表单参数同名了，测试结果显示使用的是 url 参数。
     ![Postman3.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7brrsqd1jj30ti0edaau.jpg)

2. 方法参数获取
   这个和 get 方法也没啥区别。

3. RequestParam 注解方法
   和上面两种方式略有不同，当 post 表单的参数和 url 参数同名时，会合并成一个字符串：
   ![Postman4.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7brzgy69cj30t40ee74z.jpg)

4. Bean 方式
   这种方式不区分 get，post。如果是复杂的交互接口，完全可以考虑用 bean 的方式来定义请求参数。

   ```bash
   # 输出：user:ControllerTest.User(user=dai, password=abc)
   ```

## 补充

还有一个注解需要补充下：@RequestBody

`@RequestBody`注解常用来处理`content-type`不是默认的`application/x-www-form-urlcoded`编码的内容，比如说：`application/json`或者是`application/xml`等。一般情况下来说常用其来处理`application/json`类型。另外，`@RequestBody` 通常要求调用方使用 post 请求。

需要注意的是 Controller 方法中的参数只能包含一个 @RequestBody，看下面的例子：

```java
@Data
private static class User {
    private String user;
    private String password;
}

@PostMapping(value = "req6")
public String req6(@RequestBody User user) {
    return "req6_user:" + user;
}
```

Postman 中选择 Headers 的`Content-Type`为`application/json`，在 Body 中选择发送 JSON 格式的数据。

![Postman5.png](http://ww1.sinaimg.cn/large/005Mjp9lly1g7bt5hw0uzj30t90e974z.jpg)

有关 @RequestBody 详细的介绍可以看这边博文：[Spring之RequestBody的使用姿势小结](https://juejin.im/post/5b5efff0e51d45198469acea#heading-11)。

## 参考资料

1. [HTTP菜鸟教程](https://www.runoob.com/http/http-intro.html)
2. [REST模式中HTTP请求方法（GET，POST,PUT,DELETE）](https://my.oschina.net/airship/blog/3035621)
3. [HTTP｜GET 和 POST 区别？网上多数答案都是错的！](https://juejin.im/entry/597ca6caf265da3e301e64db)（深度好文）
4. [99%的人理解错 HTTP 中 GET 与 POST 的区别](https://www.oschina.net/news/77354/http-get-post-different)
5. [SpringMVC之请求参数的获取方式]([https://blog.hhui.top/hexblog/2018/01/04/SpringMVC%E4%B9%8B%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0%E7%9A%84%E8%8E%B7%E5%8F%96%E6%96%B9%E5%BC%8F/#SpringMVC%E4%B9%8B%E8%AF%B7%E6%B1%82%E5%8F%82%E6%95%B0%E7%9A%84%E8%8E%B7%E5%8F%96%E6%96%B9%E5%BC%8F](https://blog.hhui.top/hexblog/2018/01/04/SpringMVC之请求参数的获取方式/#SpringMVC之请求参数的获取方式))
6. [Spring之RequestBody的使用姿势小结](https://juejin.im/post/5b5efff0e51d45198469acea#heading-11)