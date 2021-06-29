---
title: Mybatis与Spring整合时的注意点。
date: 2019-07-19 13:48:15
tags: Mybatis
categories: Mybatis
description: 基础补充
---

在`Mybatis`整合`Spring`的时候，有几个点需要注意：

1. `SqlSessionFactory`对象应该放到`Spring`容器中作为单例存在。
2. 用传统的`dao`开发方式时，从`Spring`容器中获得`SqlSession`对象。
3. 用`Mapper`代理开发方式时，从`Spring`容器中直接获得`mapper`的代理对象。
4. 数据库的连接及数据库连接池事务的配置都交给`Spring`容器来完成。

## 整合时DAO的两种开发方式

对于`DAO`的开发，有两种方式。

1. 传统`DAO`的开发方式；
2. 使用`Mapper`代理形式开发方式
   - 直接配置`Mapper`代理；
   - 使用扫描包配置`Mapper`代理。

### 传统的DAO开发

传统`DAO`的开发方式需要使用“接口+实现类”来完成。`DAO`的实现类需要继承`SqlSessionDaoSupport`类。
`User.xml`和`UserDAO`的代码就省略了，前者写的是`Sql`语句，后者是一个接口。这两个写完后，你需要编写接口的实现类`UserDAOImpl.java`：

```java
package hdu.dqj.mybatis.dao.impl;

import hdu.dqj.mybatis.dao.UserDao;
import hdu.dqj.mybatis.pojo.User;
import org.apache.ibatis.session.SqlSession;
import org.mybatis.spring.support.SqlSessionDaoSupport;

import java.util.List;

// 注意这里需要继承 SqlSessionDaoSupport 类
public class UserDaoImpl extends SqlSessionDaoSupport implements UserDao {
    @Override
    public User getUserById(Integer id) {
        // SqlSessionDaoSupport提供了getSqlSession()方法来获取SqlSession
        SqlSession sqlSession = super.getSqlSession();
        // 使用SqlSession来操作数据库
        User user = sqlSession.selectOne("user.getUserById", 1);
        return user;
    }
}
```

接着你还需要在`SqlMapConfig.xml`中配置`User.xml`文件：

```xml
<configuration>
    <mappers>
        <mapper resource="hdu/dqj/mybatis/pojo/user.xml" />
    </mappers>
</configuration>
```

最后把`DAO`实现类配置到`Spring`容器中：

```xml
<!-- 传统dao配置，配置dao到spring中 -->
<bean class="hdu.dqj.mybatis.dao.impl.UserDaoImpl">
    <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

### Mapper代理方式开发DAO

同样，首先需要编写`UserMapper.xml`和接口`UserMapper.java`。用`Mapper`代理的方式开发`DAO`，不需要编写接口的实现类（因为有类自动帮我们生成接口的代理），也不需要在`SqlMapConfig.xml`中配置映射文件`UserMapper.xml`（因为与`Spring`整合后，`Spring`提供了`mapperLocations`属性来扫描指定路径下的映射文件）。但是你有可能在`SqlMapConfig.xml`中配置一些别名，当然这一步也可以在`applicationContext.xml`中的`SqlSessionFactory`里面进行配置。

在`SqlMapConfig.xml`中配置别名：

```xml
<configuration>
    <!-- 别名标签，给实体类定义别名 -->
    <typeAliases>
        <!-- 别名包扫描器(推荐使用此方式)，整个包下的类都被定义别名，别名为类名，不区分大小写 -->
        <package name="hdu.dqj.mybatis.pojo" />
    </typeAliases>
</configuration>
```

在`applicationContext.xml`中配置别名：

```xml
<!-- SqlSessionFactory的配置 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource" />
    <!-- 加载mybatis核心配置文件 -->
    <property name="configLocation" value="classpath:SqlMapConfig.xml" />
    <!-- 使用mapperLocations加载扫描包文件 -->
    <property name="mapperLocations" value="classpath:mapper/*Mapper.xml" />
    <!-- 别名包扫描 -->
    <property name="typeAliasesPackage" value="hdu.dqj.mybatis.pojo" />
</bean>
```

前面说过可以直接配置`Mapper`代理，也可以使用扫描包配置`Mapper`代理。下面来看下这两种使用情况：

1. 直接配置`Mapper`代理的方式
   在`applicationContext.xml`中添加`MapperFactoryBean`的配置。在`Mybatis`和`Spring`集成时，可以使用`MapperFactoryBean`来生成`Mapper`接口的代理对象。

   ```xml
   <bean id="baseMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
       <!-- 配置Mapper接口 -->
       <property name="mapperInterface" value="hdu.dqj.mybatis.mapper.UserMapper" />
       <!-- 配置sqlSessionFactory -->
       <property name="sqlSessionFactory" ref="sqlSessionFactory" />
   </bean>
   ```

   `MapperFactoryBean`创建的代理类实现了`UserMapper`接口，并且注入到应用程序中。因为代理创建在运行时环境中，那么指定的映射器必须是一个接口，而不是一个具体的实现类。

   上述配置方式的缺点也很明显，假如我有多个接口需要使用代理的方式来实现，那么就得配置多个`mapperInterface`。这时就可以使用第二种方式。

2. **使用扫描包配置`Mapper`代理(更好)**
   没有必要在`Spring`的`XML`配置文件中注册所有的映射器，可以使用`org.mybatis.spring.mapper.MapperScannerConfigurer`这个类，**它会查找类路径下的映射器并自动将它们创建成MapperFactoryBean**。

   ```xml
   <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
       <!-- basePackage 属性是让你为映射器接口文件设置基本的包路径。你可以使用分号或逗号作为分隔符设置多于一个的包路径。 -->
       <property name="basePackage" value="hdu.dqj.mybatis.mapper" />
   </bean>
   ```

   注意 , 没有必要去指定 `SqlSessionFactory` 或 `SqlSessionTemplate` , 因为 `MapperScannerConfigurer` 将会创建 `MapperFactoryBean`，之后自动装配。但是，如果你使用了一个以上的 `DataSource`，那么自动装配可能会失效 。这种 情况下，你可以使用 `sqlSessionFactoryBeanName` 或 `sqlSessionTemplateBeanName` 属性来设置正确的 `bean` 名称来使用。这就是它如何来配置的。注意 `bean` 的名称是必须的，而不是 `bean` 的引用。因 此，`value` 属性在这里替代通常的`ref`：

   ```xml
   <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
   ```

## 参考资料

[Mybatis MapperScannerConfigurer 自动扫描 将Mapper接口生成代理注入到Spring](https://www.cnblogs.com/daxin/p/3545040.html)

