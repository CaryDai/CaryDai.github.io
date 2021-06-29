---
title: Spring学习记录(1)
date: 2019-06-06 13:40:50
categories: Spring
tags: Spring
---

Spring：SE/EE 开发的一站式框架。

- WEB层：SpringMVC
- Service层：Spring的Bean管理，Spring声明式事务。
- DAO层：Spring的JDBC模板，Spring的ORM模块。

## IOC

IOC：Inversion of Control(控制反转)，将对象的创建权交给Spring。

Q：为什么要这样做呢？

首先来看一个例子，比如现在有一个用户管理的DAO接口和对应的实现类：

```java
/**
 * 用户管理的业务层接口
 */
public interface UserDao {
    void save();
}
```

```java
/**
 * 用户管理业务层实现类
 */
public class UserDaoImpl implements UserDao {
    private String name;

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void save() {
        System.out.println("UserService执行了。。。");
    }
}
```

在使用这个接口时，代码这样写：

```java
import org.junit.Test;

public class SpringDemo1 {
    @Test
    /**
     * 传统方式的调用
     */
    public void demo1() {
        UserDaoImpl userDao = new UserDaoImpl();
        userDao.setName("王东");
        userDao.save();
    }
}
```

可是如果底层的实现切换了，就需要修改源码，比如接口换了，那得写另一份使用代码。能不能在不修改源代码的情况下对程序进行扩展呢？这时就可以用IOC了。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%9B%BE%E4%B8%80%20SpringIOC%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0.bmp)

通过`工厂模式+反射+配置文件`的方式，实现了程序解耦合。

将实现类交给Spring管理，需要编写Spring配置文件`applicationContext.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 引入约束，spring用的是schema约束，不是dtd约束 -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- Spring的入门配置 -->
    <bean id="userDAO" class="hdu.dqj.spring.demo1.UserDaoImpl">
        <!-- Spring在创建上面这个类的实例过程中，发现这个类里面有一个属性(依赖)，就通过下面的配置把属性设置进来 -->
        <property name="name" value="李东" />
    </bean>
</beans>
```

此时，我们就能用Spring的方式进行接口的调用了。

```java
@Test
    /**
     * Spring方式的调用
     */
    public void demo2() {
        // 创建Spring的工厂
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        UserDao userDao = (UserDao) applicationContext.getBean("userDAO");	//和bean标签里的id相同
        userDao.save();
    }
```

## DI

DI：依赖注入，前提必须有 IOC 的环境，依赖注入包含在IOC下。Spring 管理这个类的时候，将类的依赖属性注入（设置）进来。如上面的`userDao`类，它的`name`属性就通过`<property>`标签进行注入，这里其实用的是`Set方法`注入。

### 什么是依赖注入？

对象的生成不再是通过显示的`new`，而是直接到Spring容器里去获取。对象的创建是使用“注入”这种方式。

### 为什么需要依赖注入？

主要是为了程序解耦。

## Spring的工厂类

- BeanFactory：老版本的工厂类，调用`getBean`的时候，才会生成类的实例。
- ApplicationContext：新版本的工厂类，加载配置文件的时候，就会将`Spring`管理的类都实例化。它有两个实现类：
  - **ClassPathXmlApplicationContext**：加载类路径下的配置文件。
  - FileSystemXmlApplicationContext：加载文件系统下的配置文件。

## Bean管理

### Bean的生命周期的配置

- init-method：Bean被初始化的时候执行的方法。
- destroy-method：Bean被销毁的时候执行的方法。

### Bean的作用范围的配置

scope：Bean的作用范围。

- **singleton**：默认的，Spring会采用单例模式创建这个对象。
- **prototype**：多例模式。
- request：应用在`web`项目中，Spring创建这个类以后，将这个类存入到`request`范围中。
- session：应用在`web`项目中，Spring创建这个类以后，将这个类存入到`session`范围中。
- globalsession：应用在`web`项目中，必须在`porlet`环境下使用。但是如果没有这个环境，就相当于session。

上述两种配置方式如下：

```xml
<bean id="CustomerDAO" class="hdu.dqj.spring.demo2.CustomerDAOImpl" scope="singleton"
          init-method="setup" destroy-method="destroy" />
```

### Bean的实例化方式

Spring 在实例化类的时候，有如下几种方式：

1. 使用无参构造方法（默认）

2. 静态工厂实例化的方式：
   首先，编写一个类（比如`Car`）的静态工厂。

   ```java
   public class CarFactory {
       public static Car createCar() {
           System.out.println("CarFactory中的方法执行了。。。");
           return new Car();
       }
   }
   ```

   接着，在配置文件`applicationContext.xml`中配置即可：

   ```xml
   <!-- 静态工厂实例化 -->
   <bean id="car" class="hdu.dqj.spring.demo2.CarFactory" factory-method="createCar" />
   ```

3. 实例工厂实例化的方式
   首先，编写一个类（比如`Phone`）的实例工厂。

   ```java
   public class PhoneFactory {
       public Phone createPhone() {
           System.out.println("Phone的实例工厂执行了。。。");
           return new Phone();
       }
   }
   ```

   接着，在配置文件中配置：

   ```xml
   <bean id="phoneFactory" class="hdu.dqj.spring.demo2.PhoneFactory"  />
   <bean id="phone" factory-bean="phoneFactory" factory-method="createPhone" />
   ```

### Spring的属性注入

1. **构造方法的属性注入**

   ```java
   // 定义Car类
   public class Car {
       private String name;
       private Double price;
       public Car(String name, Double price) {
           this.name = name;
           this.price = price;
       }
   }
   ```

   配置文件中这样写：

   ```xml
   <bean id="car" class="hdu.dqj.spring.demo3.Car">
       <constructor-arg name="name" value="宝马" />
       <constructor-arg name="price" value="800000" />
   </bean>
   ```

2. **Set方法的属性注入**
   第一个例子中有写。配置文件中使用`<property>`标签：

   ```xml
   <bean id="car2" class="hdu.dqj.spring.demo3.Car2">
       <property name="name" value="奔驰" />
       <property name="price" value="1000000" />
   </bean>
   ```

   当设置对象类型的属性时，则需要用到`ref`参数：

   ```java
   public class Employee {
       private String name;
       private Car2 car2;
       public void setName(String name) {
           this.name = name;
       }
       public void setCar2(Car2 car2) {	//对象类型参数car2
           this.car2 = car2;
       }
   }
   ```

   ```xml
   <bean id="employee01" class="hdu.dqj.spring.demo3.Employee">
       <!-- value:设置普通类型的值，ref：设置其他的类的id或name -->
       <property name="name" value="涛哥" />
       <!-- ref的car2对应上面的id -->
       <property name="car2" ref="car2" />
   </bean>
   ```

3. **SpEL的属性注入**

   SpEL：Spring Expression Language，Spring的表达式语言。语法：`#{SpEL}`。

   ```xml
   <!-- SpEL的属性注入 -->
   <bean id="carInfo" class="hdu.dqj.spring.demo3.CarInfo" />
   
   <bean id="car2" class="hdu.dqj.spring.demo3.Car2">
    	<!-- 下面这个语句会调用CarInfo类里的getName()方法 -->
       <property name="name" value="#{carInfo.name}" />
       <property name="price" value="#{carInfo.calculatePrice()}" />
   </bean>
   
   <bean id="employee02" class="hdu.dqj.spring.demo3.Employee">
       <property name="name" value="#{'赵洪'}" />
       <property name="car2" value="#{car2}" />
   </bean>
   ```

## IOC 的注解开发

要想使用注解开发，首先应该引入context约束：`xmlns:context="http://www.springframework.org/schema/context"`。其次，还应该扫描类上的注解：

```xml
<!-- 使用IOC的注解开发，配置组件扫描（哪些包下的类使用IOC的注解） -->
<!-- 扫描父包也会扫描子包。扫描是为了扫描类上的注解 -->
<context:component-scan base-package="hdu.dqj.spring" />
```

接着，编写一个类，并在类上面加上注解：

```java
/**
 * 用户DAO的实现类
 */
@Component("userDao")   // 相当于<bean id="userDao" class="hdu.dqj.spring.demo1.UserDaoImpl" />
public class UserDaoImpl implements UserDao {
    private String name;
    
    @Value("王东")
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void save() {
        System.out.println("Dao中保存用户的方法执行了。。。" + name);
    }
}
```

在使用时和之前的写法类似：

```java
@Test
// Spring的IOC的注解方式
public void demo2() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
    UserDao userDao = (UserDao) applicationContext.getBean("userDao222");
    userDao.save();
}
```

使用注解方式，可以没有set方法。

- 属性如果有 set 方法，需要将属性注入的注解添加到 set 方法；
- 属性如果没有 set 方法，需要将属性注入的注解添加到属性上。

### IOC的注解详解

- 类上的注解
  @Component：组件。修饰一个类，将这个类交给Spring管理。这个注解有三个衍生注解（功能类似），修饰类：

  - **@Controller：web层**
  - **@Service：Service层**
  - **@Repository：dao层**

- 属性注入的注解

  - 普通属性：**@Value**

  - 对象类型属性：

    - @Autowired：设置对象类型的属性的值。但是按照类型完成属性的注入。
      - 我们习惯上是按照名称完成属性注入：必须让 @Autowired 注解和 @Qualifier 注解一起使用完成按照名称属性注入。
    - **@Resource：完成对象类型的属性注入，按照名称完成属性注入。**

    ```java
    @Service("userService")
    public class UserServiceImpl implements UserService {
    
        // 注入Dao
    //    @Autowired
    //    @Qualifier("userDao222")
        @Resource(name = "userDao222")	//这里的userDao222是UserDaoImpl类上的注解
        private UserDao userDao;
    
        @Override
        public void save() {
            System.out.println("UserService的save方法执行了。。。");
            userDao.save();
        }
    }
    ```

- 生命周期与作用范围的注解

  - **生命周期相关注解**
    - @PostConstruct：初始化方法
    - @PreDestroy：销毁方法
  - **作用范围相关注解**
    @Scope：作用范围。取值如下：
    - **singleton：默认单例**
    - **prototype：多例**
    - request
    - session
    - globalsession

  ```java
  @Service("customerService")
  @Scope  // Bean的作用范围的注解
  public class CustomerService {
  
      @PostConstruct  // 相当于init-method
      public void init() {
          System.out.println("CustomerService被初始化了。。。");
      }
  
      public void save() {
          System.out.println("Service的save方法执行了。。。");
      }
  
      @PreDestroy     // 相当于destroy-method
      public void destroy() {
          System.out.println("CustomerService被销毁了。。。");
      }
  }
  ```

  ```java
  @Test
  public void demo1() {
      ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
      CustomerService customerService = (CustomerService) applicationContext.getBean("customerService");
      System.out.println(customerService);
      applicationContext.close();     // 调用close方法才会销毁
  }
  ```

### IOC 的 XML和注解开发的比较

|                        | 基于XML配置                                  | 基于注解配置                                                 |
| ---------------------- | -------------------------------------------- | ------------------------------------------------------------ |
| Bean定义               | 使用`Bean`标签，定义`id`属性和`class`属性    | @Component  衍生类@Controller @Service @Repository           |
| Bean名称               | 通过 id 或 name 指定                         | @Component("person")                                         |
| Bean注入               | 构造方法、Setter方法或者通过SpEL             | @Autowired 按类型注入  @Qualifier 按名称注入  @Resource      |
| 生命过程、Bean作用范围 | init-method、destroy-method  范围：scope属性 | @PostConstruct 初始化  @PreDestroy 销毁  @Scope 设置作用范围 |
| 适合场景               | Bean来自第三方                               | Bean的实现类由用户自己开发                                   |

### XML 和注解整合开发

XML 管理 Bean，注解完成属性注入。

`applicationContext.xml`里面：

```xml
<bean id="productService" class="hdu.dqj.spring.demo3.ProductService" />
<bean id="productDao" class="hdu.dqj.spring.demo3.ProductDao" />
<bean id="orderDao" class="hdu.dqj.spring.demo3.OrderDao" />
```

`ProductService.java`：

```java
public class ProductService {

    @Resource(name = "productDao")
    private ProductDao productDao;
    @Resource(name = "orderDao")
    private OrderDao orderDao;

    public void save() {
        System.out.println("ProductService的save方法执行了。。。");
        productDao.save();
        orderDao.save();
    }
}
```

