---
title: Spring整合Mybatis单元测试
date: 2019-07-19 10:32:11
tags: 
	- Spring
	- Mybatis
categories: Spring
---

最近在写SSM框架的练手项目，在进行`Spring`和`Mybatis`整合时，写单元测试的过程中，遇到了很多问题。有必要记录下，因为这种单元测试应该会经常用到。

## 以前的测试方法

最早学这两个框架整合的时候，没有用到`Spring`整合`JUnit4`测试的方式。
当时在`Mybatis`的配置文件中写了别名包扫描（因为在`UserMapper.xml`中写`select`标签时，里面的`resultType`没有写包名+类名的方式）和映射文件包扫描：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//hdu.dqj.mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!-- dtd要求必须严格按照标签的顺序来写 -->
<configuration>
    <!-- 别名标签，给实体类定义别名 -->
    <typeAliases>
        <!-- 别名包扫描器(推荐使用此方式)，整个包下的类都被定义别名，别名为类名，不区分大小写 -->
        <package name="hdu.dqj.mybatis.pojo" />
    </typeAliases>

    <mappers>
        <!--<mapper resource="hdu/dqj/mybatis/pojo/user.xml" />-->
        <!--
            映射文件包扫描（推荐方式）：
            1、接口文件必须与映射文件处于同一目录下
            2、接口文件的名称必须与映射文件的名称一致(都是UserMapper)
        -->
        <package name="hdu.dqj.mybatis.mapper" />
    </mappers>
</configuration>
```

在`Spring`的配置文件`applicationContext.xml`中配置数据源、SqlSessionFactory、包扫描的位置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 加载数据库配置文件 -->
	<context:property-placeholder location="classpath:jdbc.properties" />

	<!-- 数据库连接池 -->
	<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
		<property name="driverClassName" value="${jdbc.driver}" />
		<property name="url" value="${jdbc.url}" />
		<property name="username" value="${jdbc.username}" />
		<property name="password" value="${jdbc.password}" />
		<!-- 连接池的最大数据库连接数 -->
		<property name="maxIdle" value="10" />
		<!-- 最大空闲数 -->
		<property name="minIdle" value="5" />
	</bean>

	<!-- SqlSessionFactory的配置 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<!-- 加载mybatis核心配置文件 -->
		<property name="configLocation" value="classpath:SqlMapConfig.xml" />
		<!-- 别名包扫描，SqlMapConfig.xml中已经写了 -->
		<!--<property name="typeAliasesPackage" value="hdu.dqj.mybatis.mapper" />-->
	</bean>

	<!-- 动态代理第二种方式：包扫描(推荐) -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="hdu.dqj.mybatis.mapper" />
	</bean>
</beans>
```

这里如果没有写最下面的那对`bean`，在之后的测试中将会报错：`org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'hdu.dqj.mybatis.mapper.UserMapper' available`。

之后编写测试文件`UserMapperTest.java`：

```java
package hdu.dqj.mybatis.mapper;

import hdu.dqj.mybatis.pojo.User;
import org.junit.Before;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class UserMapperTest {

    private ApplicationContext applicationContext;

    @Before
    // 在执行其它代码之前，先执行这段代码
    public void init() {
        applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    }

    @Test
    public void testGetUserById() {
        // 如果没有之前的包扫描，这里就无法实例化UserMapper。因为Spring找不到UserMapper接口。
        UserMapper userMapper = applicationContext.getBean(UserMapper.class);
        User user = userMapper.getUserById(30);
        System.out.println(user);
    }
}
```

## 使用JUnit4整合Spring的测试方法

而使用`JUnit4`整合`Spring`的方式进行单元测试时，就方便许多：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring/applicationContext-dao.xml"})
public class DepartmentMapperTest {
    @Autowired
    private DepartmentMapper departmentMapper;

    @Test
    public void selectDeptByIdTest() {
        Department department = departmentMapper.selectDeptById(1);
        System.out.println(department);
    }
}
```

- `@RunWith` 注释标签是 `Junit` 提供的，用来说明此测试类的运行者，这里用了 `SpringJUnit4ClassRunner`，这个类是一个针对 `Junit` 运行环境的自定义扩展，用来标准化在 `Spring`环境中 `Junit4.5` 的测试用例，例如支持的注释标签的标准化。

- `@ContextConfiguration` 注释标签是 `Spring test context` 提供的，用来指定 `Spring` 配置信息的来源，支持指定 XML 文件位置或者 `Spring` 配置类名，这里配置了`classpath`下的`spring/applicationContext-dao.xml`为配置文件。这个配置文件里面需要使用`mapperLocations`指明加载扫描映射文件包的位置：

  ```xml
  <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
      <!-- 设置MyBatis核心配置文件 -->
      <property name="configLocation" value="classpath:SqlMapConfig.xml" />
      <!-- 设置数据源 -->
      <property name="dataSource" ref="dataSource"/>
      <!-- 下面的配置暂时为了进行单元测试 -->
      <!-- 加载扫描包文件 -->
      <property name="mapperLocations" value="classpath:mapper/*Mapper.xml" />
  </bean>
  ```

- `@Autowired` 体现了我们的测试类也是在 `Spring` 的容器中管理的，他可以获取容器的 `bean` 的注入，我们不用自己手工获取要测试的 `bean` 实例了。

## 参考资料

[使用Spring进行单元测试](https://www.ibm.com/developerworks/cn/java/j-lo-springunitest/index.html)