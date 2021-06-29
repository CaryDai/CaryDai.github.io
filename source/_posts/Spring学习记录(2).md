---
title: Spring学习记录(2)
date: 2019-06-11 19:54:08
categories: Spring
tags: Spring
---

## AOP开发

### XML方式的AOP开发

在软件业，AOP为Aspect Oriented Programming的缩写，意为：[面向切面编程](https://baike.baidu.com/item/%E9%9D%A2%E5%90%91%E5%88%87%E9%9D%A2%E7%BC%96%E7%A8%8B/6016335)，通过[预编译](https://baike.baidu.com/item/%E9%A2%84%E7%BC%96%E8%AF%91/3191547)方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是[OOP](https://baike.baidu.com/item/OOP)的延续，是软件开发中的一个热点，也是[Spring](https://baike.baidu.com/item/Spring)框架中的一个重要内容，是[函数式编程](https://baike.baidu.com/item/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B/4035031)的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的[耦合度](https://baike.baidu.com/item/%E8%80%A6%E5%90%88%E5%BA%A6/2603938)降低，提高程序的可重用性，同时提高了开发的效率。（来自百度百科）

比如一开始是需要在数据保存到数据库前，对每一个dao都要进行权限的校验，此时可以生成一个代理，平时都用这个代理。到后来不需要权限校验了，那么就直接调用 UserDao 的save方法即可。

#### AOP底层实现原理

动态代理

- JDK 动态代理：只能对实现了接口的类产生代理。
- Cglib 动态代理（类似于 Javassist 第三方代理技术）：对没有实现接口的类产生代理对象。生成子类对象。

#### AOP 开发中相关术语

![](https://ws1.sinaimg.cn/large/005Mjp9lgy1g3zl63bfibj311y0lch41.jpg)

#### AOP 的入门开发（AspectJ 的 XML 方式）

AspectJ是一个面向切面的框架，它扩展了Java语言。AspectJ定义了AOP语法，它有一个专门的[编译器](https://baike.baidu.com/item/编译器/8853067)用来生成遵守Java字节编码规范的Class文件。

1. 首先需要引入aop开发相关的jar包，在`applicationContext.xml`中引入AOP约束：
   ![](https://ws1.sinaimg.cn/large/005Mjp9lly1g3zl9zgdomj30d8032mx5.jpg)

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:aop="http://www.springframework.org/schema/aop"
          xsi:schemaLocation="
               http://www.springframework.org/schema/beans
               http://www.springframework.org/schema/beans/spring-beans.xsd
               http://www.springframework.org/schema/aop
               http://www.springframework.org/schema/aop/spring-aop.xsd">
   
   </beans>
   ```

2. 接着编写一个切面类

   ```java
   /**
    * 切面类
    */
   public class MyAspectXML {
       public void chechPri() {
           System.out.println("权限校验===========");
       }
   }
   ```

3. 将切面类交给Spring管理

   ```xml
   <!-- 将切面类交给Spring管理 -->
   <bean id="myAspect" class="hdu.dqj.spring.demo3.MyAspectXML" />
   ```

4. 通过AOP的配置完成对目标类产生代理

   ```xml
   <aop:config>
       <!-- 配置切入点，表达式配置哪些类的哪些方法需要进行增强 -->
       <aop:pointcut id="pointcut1" expression="execution(* hdu.dqj.spring.demo3.ProductDaoImpl.save(..))" />
   
       <!-- 配置切面(通知和切入点的组合) -->
       <aop:aspect ref="myAspect">
           <aop:before method="checkPri" pointcut-ref="pointcut1" />
       </aop:aspect>
   </aop:config>
   ```

5. 编写测试类，Spring 整合 JUnit 单元测试，需要引入 spring-test-5.1.5.RELEASE.jar 包

   ```java
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration("classpath:applicationContext.xml")
   public class SpringDemo3 {
   
       @Resource(name = "productDao")  // 属性注入
       private ProductDao productDao;
   
       @Test
       public void demo1() {
           productDao.save();
           productDao.update();
           productDao.find();
           productDao.delete();
       }
   }
   ```

   当然，在`applicationContext.xml`中需要事先指明被增强的对象：

   ```xml
   <!-- 配置目标对象：被增强的对象 -->
   <bean id="productDao" class="hdu.dqj.spring.demo3.ProductDaoImpl" />
   ```

#### Spring中通知的类型

- 前置通知：在目标方法执行之前进行操作
  可以获得切入点的信息。

  ```xml
  <!-- 配置切面(通知和切入点的组合) -->
  <aop:aspect ref="myAspect">
      <!-- 前置通知======= -->
      <aop:before method="checkPri" pointcut-ref="pointcut1" />
  </aop:aspect>
  ```

- 后置通知：在目标方法执行之后进行操作
  获得方法的返回值。

  ```xml
  <!-- 后置通知======= -->
  <!-- returning的值和writeLog传的参数值相同 -->
  <aop:after-returning method="writeLog" pointcut-ref="pointcut2" returning="result" />
  ```

- 环绕通知：在目标方法执行前后进行的操作

  ```xml
  <!-- 环绕通知======== -->
  <aop:around method="around" pointcut-ref="pointcut3" />
  ```

- 异常抛出通知：在程序出现异常的时候，进行的操作

  ```xml
  <!-- 异常抛出通知====== -->
  <aop:after-throwing method="afterThrowing" pointcut-ref="pointcut4" throwing="ex" />
  ```

- 最终通知：无论代码是否有异常，总是会执行。

  ```xml
  <!-- 最终通知 -->
  <aop:after method="after" pointcut-ref="pointcut4" />
  ```

修改后的切面类：

```java
package hdu.dqj.spring.demo3;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;

/**
 * 切面类
 */
public class MyAspectXML {
    /**
     * 前置通知
     * @param joinPoint
     */
    public void checkPri(JoinPoint joinPoint) {
        System.out.println("权限校验===========" + joinPoint);
    }

    /**
     * 后置通知
     */
    public void writeLog(Object result) {
        System.out.println("日志记录===========" + result);
    }

    /**
     * 性能监控
     * @param joinPoint
     * @return
     * @throws Throwable
     */
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("环绕前通知============");
        // 这句话相当于执行目标程序。对照配置文件，可以发现就相当于执行了update()方法
        Object obj = joinPoint.proceed();
        System.out.println("环绕后通知============");
        return obj;
    }

    /**
     * 异常抛出通知
     */
    public void afterThrowing(Throwable ex) {
        System.out.println("异常抛出通知=========" + ex.getMessage());
    }

    /**
     * 最终通知：相当于finally代码块中的内容
     */
    public void after() {
        System.out.println("最终通知=========");
    }
}
```

#### Spring的切入点表达式写法

基于`execution`函数完成。语法：[访问修饰符] 方法返回值 包名.类名.方法名(参数)

```
execution(public void hdu.dqj.spring.CustomerDao.save(..))       // ..  表示任意参数
execution(* *.*.*.*Dao.save(..))	// *是通配符
execution(* hdu.dqj.spring.CustomerDao+.save(..))	//CustomerDao+表示这个类和其子类
execution(* hdu.dqj.spring..*.*(..))	// spring包的子包下的所有类的所有方法
```

### 基于 AspectJ 注解开发的 AOP 开发方式

1. 编写并配置目标类（被增强类）

   ```java
   <!-- 配置目标类 -->
   <bean id="targetOrderDao" class="hdu.dqj.spring.demo1.OrderDao" />
   ```

2. 编写并配置切面类

   ```java
   public class MyAspectAnno {
   	public void before() {
           System.out.println("前置增强============");
       }
   }
   ```
   ```xml
   <!-- 配置切面类 -->
	<bean id="myAspect" class="hdu.dqj.spring.demo1.MyAspectAnno" />
   ```
   
3. 使用注解的 AOP 对目标类进行增强
   首先在配置文件中打开注解的 AOP 开发

   ```xml
   <!-- 在配置文件中开启注解的AOP开发 -->
   <aop:aspectj-autoproxy />
   ```

   接着在切面类上使用注解

   ```java
   @Aspect
   public class MyAspectAnno {
   	@Before(value = "execution(* hdu.dqj.spring.demo1.OrderDao.save(..))")
   	public void before() {
           System.out.println("前置增强============");
       }
   }
   ```

4. 编写测试类

   ```java
   /**
    * Spring的AOP的注解开发
    */
   // 下面两个注解是整合Spring和Junit测试
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration("classpath:applicationContext.xml")
   public class SpringDemo1 {
   
       @Resource(name = "targetOrderDao")
       private OrderDao orderDao;
   
       @Test
       public void demo1() {
           orderDao.save();
           orderDao.update();
           orderDao.find();
           orderDao.delete();
       }
   }
   ```

至此，`save()`方法就被增强了。下面的代码展示了注解 AOP 的**通知类型**和**切入点配置**的写法：

```java
package hdu.dqj.spring.demo1;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

/**
 * 切面类：注解的切面类
 */
@Aspect
public class MyAspectAnno {

    @Before(value = "execution(* hdu.dqj.spring.demo1.OrderDao.save(..))")
    public void before() {
        System.out.println("前置增强============");
    }

    // 后置通知
    @AfterReturning(value = "MyAspectAnno.pointcutDelete()", returning = "result")
    public void afterReturning(Object result) {
        System.out.println("后置增强===========" + result);
    }

    // 环绕通知
    @Around(value = "MyAspectAnno.pointcutUpdate()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("环绕前增强=========");
        Object obj = joinPoint.proceed();
        System.out.println("环绕后增强=========");
        return obj;
    }

    // 异常抛出通知
    @AfterThrowing(value = "MyAspectAnno.pointcutFind()", throwing = "e")
    public void afterThrowing(Throwable e) {
        System.out.println("异常抛出增强==========" + e.getMessage());
    }

    // 最终通知
    @After(value = "MyAspectAnno.pointcutFind()")
    public void after() {
        System.out.println("最终增强=========");
    }

    // 切入点注解
    @Pointcut(value = "execution(* hdu.dqj.spring.demo1.OrderDao.find(..))")
    private void pointcutFind(){}  // 这个方法没有实际意义，只是给上面的表达式取名为pointcutFind

    @Pointcut(value = "execution(* hdu.dqj.spring.demo1.OrderDao.save(..))")
    private void pointcutSave(){}  // 这个方法没有实际意义，只是给上面的表达式取名为pointcutSave

    @Pointcut(value = "execution(* hdu.dqj.spring.demo1.OrderDao.update(..))")
    private void pointcutUpdate(){}  // 这个方法没有实际意义，只是给上面的表达式取名为pointcutUpdate

    @Pointcut(value = "execution(* hdu.dqj.spring.demo1.OrderDao.delete(..))")
    private void pointcutDelete(){}  // 这个方法没有实际意义，只是给上面的表达式取名为pointcutDelete
}
```

## Spring的JDBC模板的使用

### 使用JDBC的模板

```java
package hdu.dqj.spring.demo2;

import org.junit.Test;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

/**
 * JDBC模板的使用
 */
public class JdbcDemo1 {

    @Test
    // jdbc模板的使用类似于Dbutils
    public void demo1() {
        // 创建连接池(DriverManagerDataSource是Spring内置的连接池)
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql:///spring5_day03");
        dataSource.setUsername("root");
        dataSource.setPassword("root");
        // 创建jdbc模板
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        jdbcTemplate.update("insert into account values (null,?,?)", "小明", 20000d);
    }
}
```

### 将连接池和模板交给Spring管理

在`applicationContext.xml`中进行配置：

```xml
<!-- 配置Spring内置的连接池 -->
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <!-- 属性注入 -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver" />
    <property name="url" value="jdbc:mysql:///spring5_day03" />
    <property name="username" value="root" />
    <property name="password" value="root" />
</bean>

<!-- 配置Spring的JDBC模板 -->
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource" />
</bean>
```

使用模板：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext.xml")
public class JdbcDemo2 {
    //属性注入
    @Resource(name = "jdbcTemplate")
    private JdbcTemplate jdbcTemplate;
    
    @Test
    // 保存操作
    public void demo1() {
        jdbcTemplate.update("insert into account values (null,?,?)", "何菊花", 10000d);
    }

    @Test
    // 修改操作
    public void demo2() {
        jdbcTemplate.update("update account set name = ?, money = ? where id = ?", "何巨涛", 20000d, 6);
    }

    @Test
    // 删除操作
    public void demo3() {
        jdbcTemplate.update("delete from account where id = ?", 6);
    }

    @Test
    // 查询操作
    public void demo4() {
        // 返回的是一个字符串，所以第二个参数写String.class
        String name = jdbcTemplate.queryForObject("select name from account where id = ?", String.class, 5);
        System.out.println(name);
    }
}
```

### 使用开源的数据库连接池

首先可以把连接信息抽取到一个文件`jdbc.properties`中：

```properties
jdbc.driverClass=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql:///spring5_day03
jdbc.username=root
jdbc.password=root
```

接着在`applicationContext.xml`中引入该文件：

```xml
<!-- 引入属性文件jdbc.properties -->
<!-- 第一种方式：通过一个bean标签引入（很少使用） -->
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="location" value="classpath:jdbc.properties" />
</bean>

<!-- 第二种方式：通过context标签引入 -->
<context:property-placeholder location="classpath:jdbc.properties" />
```

引入C3P0连接池jar包，配置C3P0连接池：

```xml
<!-- 配置C3P0连接池 -->
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
    <property name="driverClass" value="${jdbc.driverClass}" />
    <property name="jdbcUrl" value="${jdbc.url}" />
    <property name="user" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>

<!-- 配置Spring的JDBC模板 -->
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource" />
</bean>
```

## Spring 的事务管理

### 事务回顾

- 什么是事务？
  事务：逻辑上的一组操作，组成这组操作的各个单元，要么全都成功，要么全都失败。
- 事务的特性
  1. 原子性：事务不可分割。
  2. 一致性：事务执行前后数据完整性保持一致。
  3. 隔离性：一个事务的执行不应该受到其他事务的干扰。
  4. 持久性：一旦事务结束，数据就持久化到数据库。
- 如果不考虑隔离性将引发的安全问题
  - 读问题
    1. 脏读：一个事务读到另一个事务未提交的数据。
    2. 不可重复读：一个事务读到另一个事务已经提交的update的数据，导致一个事务中多次查询结果不一致。
    3. 虚读、幻读：一个事务读到另一个事务已经提交的insert的数据，导致一个事务中多次查询结果不一致。
  - 写问题
    - 丢失更新：事务T1读取了数据，并执行了一些操作，然后更新数据。事务T2也做相同的事，则T1和T2更新数据时可能会覆盖对方的更新，从而引起错误。
- 解决读问题
  - 设置事务的隔离级别
    1. Read uncommitted  ：未提交读，任何读问题解决不了。
    2. **Read committed**      ：已提交读，解决脏读，但是不可重复读和虚读有可能发生。
    3. **Repeatable read**      ：重复读，解决脏读和不可重复读，但是虚读有可能发生。
    4. Serializable                ：解决所有读问题。

### Spring事务管理的API

- **PlatformTransactionManager**：平台事务管理器
  是Spring管理事务的真正的对象。
- **TransactionDefinition**：事务定义信息
  用于定义事务的相关信息，隔离级别、超时信息、**传播行为**、是否只读。
- **TransactionStatus**：事务的状态
  用于记录在事务管理过程中，事务的状态的对象。

![](https://ws1.sinaimg.cn/large/005Mjp9lgy1g3zuflifaaj30q90bhabq.jpg)

Spring进行事务管理的时候，首先**平台事务管理器**根据**事务定义信息**进行事务的管理，在事务管理过程中，产生各种状态，将这些状态的信息记录到**事务状态**的对象中。

### Spring事务的传播行为

Spring提供了七种事务的传播行为：

- 保证多个操作在同一个事务中
  - **PROPAGATION_REQUIRED**：默认值，如果A中有事务，使用A中的事务；如果A没有，创建一个新的事务，将操作包含进来。
  - PROPAGATION_SUPPORTS：支持事务，如果A中有事务，使用A中的事务；如果A没有事务，不使用事务。
  - PROPAGATION_MANDATORY：如果A中有事务，使用A中的事务；如果A没有事务，抛出异常。
- 保证多个操作不在同一个事务中
  - **PROPAGATION_REQUIRES_NEW**：如果A中有事务，将A的事务挂起（暂停），创建新事务，只包含自身操作；如果A中没有事务，创建一个新事务，包含自身操作。
  - PROPAGATION_NOT_SUPPORTED：如果A中有事务，将A的事务挂起。不使用事务管理。
  - PROPAGATION_NEVER：如果A中有事务，报异常。
- 嵌套式事务
  - **PROPAGATION_NESTED**：嵌套事务，如果A中有事务，按照A的事务执行，执行完成后，设置一个保存点，执行B中的操作，如果没有异常，执行通过，如果有异常，可以选择回滚到最初始位置，也可以回滚到保存点。

### 事务管理环境搭建

1. 创建DAO接口和实现类：

   ```java
   /**
    * 转账的DAO接口
    */
   public interface AccountDao {
       void outMoney(String from, Double money);
       void inMoney(String to, Double money);
   }
   ```

   ```java
   public class AccountDaoImpl extends JdbcDaoSupport implements AccountDao {
   
    @Override
    public void outMoney(String from, Double money) {
        this.getJdbcTemplate().update("update account set money = money - ? where name = ?", money, from);
    }
   
    @Override
    public void inMoney(String to, Double money) {
        this.getJdbcTemplate().update("update account set money = money + ? where name = ?", money, to);
    }
   }
   ```

2. 创建Service接口和实现类：

   ```java
   /**
    * 转账的业务层的接口
    */
   public interface AccountService {
       void transfer(String from, String to, Double money);
   }
   ```

   ```java
   public class AccountServiceImpl implements AccountService {
       // 注入DAO
       private AccountDao accountDao;
   
       public void setAccountDao(AccountDao accountDao) {
           this.accountDao = accountDao;
       }
       
       @Override
       /**
        * from:转出的账号
        * to:转入的账号
        * money:转账的金额
        */
       public void transfer(String from, String to, Double money) {
           accountDao.outMoney(from, money);
           accountDao.inMoney(to, money);
       }
   }
   ```

3. 配置数据库连接池和JDBC模板

   ```xml
   <!-- 配置连接池和JDBC模板 -->
   <context:property-placeholder location="classpath:jdbc.properties" />
   
   <!-- 配置C3P0连接池 -->
   <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
       <property name="driverClass" value="${jdbc.driverClass}" />
       <property name="jdbcUrl" value="${jdbc.url}" />
       <property name="user" value="${jdbc.username}" />
       <property name="password" value="${jdbc.password}" />
   </bean>
   ```

4. 配置Service和DAO：交给Spring管理

   ```xml
   <!-- 配置Service -->
   <bean id="accountService" class="hdu.dqj.spring.tx.AccountServiceImpl">
       <!-- 注入accountDao -->
       <property name="accountDao" ref="accountDao" />
   </bean>
   
   <!-- 配置DAO -->
   <bean id="accountDao" class="hdu.dqj.spring.tx.AccountDaoImpl">
       <!-- 在DAO中注入JDBC模板 -->
       <property name="dataSource" ref="dataSource" />
   </bean>
   ```

### 第一类：编程式事务管理（不常用）

1. 配置平台事务管理器

   ```xml
   <!-- 配置平台事务管理器。管理器需要事务，事务从Connection获得，连接从连接池DataSource获得 -->
   <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <property name="dataSource" ref="dataSource" />
   </bean>
   ```

2. 配置事务管理模板

   ```xml
   <!-- 配置事务管理的模板 -->
   <bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
   	<property name="transactionManager" ref="transactionManager" />
   </bean>
   ```

3. 在业务层注入事务管理模板

   ```xml
   <!-- 配置Service -->
   <bean id="accountService" class="hdu.dqj.spring.tx.AccountServiceImpl">
       <property name="accountDao" ref="accountDao" />
       <!-- 注入事务管理的模板 -->
       <property name="transactionTemplate" ref="transactionTemplate" />
   </bean>
   ```

4. 编写事务管理的代码

   ```java
   public void transfer(String from, String to, Double money) {
   
       transactionTemplate.execute(new TransactionCallbackWithoutResult() {
           @Override
           protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
               accountDao.outMoney(from, money);
               int d = 1/0;
               accountDao.inMoney(to, money);
           }
       });
   }
   ```

5. 测试

   ```java
   /**
    * 测试转账的环境
    */
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration("classpath:tx.xml")
   public class SpringDemo1 {
   
       @Resource(name = "accountService")
       private AccountService accountService;
   
       @Test
       public void demo1() {
           accountService.transfer("赵冠希", "王宝强", 1000d);
       }
   }
   ```

### 第二类：声明式事务管理（AOP方式）

#### XML方式

同样需要配置事务管理器。

接着配置事务的增强和AOP的配置：

```xml
<!-- 配置事务的增强 -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!-- 事务管理的规则 -->
        <!--propagation 传播行为, REQUIRED：必须；REQUIRES_NEW:必须是新的-->
        <!--<tx:method name="save*" propagation="REQUIRED" isolation="DEFAULT" />-->
        <!--<tx:method name="update*" propagation="REQUIRED" />-->
        <!--<tx:method name="delete*" propagation="REQUIRED" />-->
        <!--<tx:method name="find*" read-only="true" />-->
        <tx:method name="*" propagation="REQUIRED" />
    </tx:attributes>
</tx:advice>

<!-- aop的配置。利用切入点表达式从目标类方法中确定增强的连接器，从而获得切入点 -->
<aop:config>
    <aop:pointcut id="pointcut1" expression="execution(* hdu.dqj.spring.tx2.AccountServiceImpl.*(..))" />
    <aop:advisor advice-ref="txAdvice" pointcut-ref="pointcut1" />
</aop:config>
```

#### 注解方式的声明式事务管理

同样先配置平台事务管理器。

接着开启注解事务：

```xml
<!-- 开启注解事务 -->
<!-- transaction-manager 配置事务管理器；proxy-target-class="true"：底层强制使用cglib 代理 -->
<tx:annotation-driven transaction-manager="transactionManager" />
```

最后在业务层添加注释：

```java
/**
 * 转账的业务层实现类
 */
@Transactional(isolation = Isolation.DEFAULT, propagation = Propagation.REQUIRED)
public class AccountServiceImpl implements AccountService {

    // 注入DAO
    private AccountDao accountDao;

    public void setAccountDao(AccountDao accountDao) {
        this.accountDao = accountDao;
    }

    @Override
    /**
     * from:转出的账号
     * to:转入的账号
     * money:转账的金额
     */
    public void transfer(String from, String to, Double money) {
        accountDao.outMoney(from, money);
        int d = 1 / 0;
        accountDao.inMoney(to, money);
    }
}
```