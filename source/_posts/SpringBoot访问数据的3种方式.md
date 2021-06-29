---
title: SpringBoot访问数据的3种方式
date: 2019-09-17 20:23:05
tags: SpringBoot
categories: SpringBoot
description: 介绍SpringBoot访问数据的三种方式
---

## 使用JDBC

首先在`pom.xml`中添加依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

接下来配置`application.yml`：

```yml
spring:
  datasource:
    username: root
    password: root
    url: jdbc:mysql://10.66.127.138:3306/jdbc?serverTimezone=GMT
    driver-class-name: com.mysql.cj.jdbc.Driver
```

在测试类里面进行测试即可，需要用到 JdbcTemplate：

```java
package com.dqj.springboot;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.junit4.SpringRunner;

import java.sql.Connection;
import java.util.List;
import java.util.Map;

@RunWith(SpringRunner.class)
@SpringBootTest
public class Springboot06DataJdbcApplicationTests {

    @Autowired
    JdbcTemplate jdbcTemplate;

    @Test
    public void testJdbc() {
        List<Map<String,Object>> mapList = jdbcTemplate.queryForList("select * from department");
        System.out.println(mapList.get(0));
    }
}
```

这样就成功使用JDBC的方式从数据库取出了数据。

## 整合MyBatis

同样，先需要在`pom.xml`中引入依赖：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```

第二步也是在`application.yml`中配置数据源：

```yml
spring:
  datasource:
    #   数据源基本配置
    username: root
    password: root
    url: jdbc:mysql://localhost:3306/jdbc?serverTimezone=GMT
    driver-class-name: com.mysql.cj.jdbc.Driver
```

接着需要先创建 JavaBean，按照数据库的字段名进行创建。

使用 MyBatis 访问数据库有两种方式，一种是注解方式，一种是配置文件方式，下面分别介绍：

### 注解方式

在`mapper`文件夹下新建`DepartmentMapper.java`接口文件，这个文件里面声明方法，在每个方法上都有注释：

```java
package com.dqj.springboot.mapper;

import com.dqj.springboot.bean.Department;
import org.apache.ibatis.annotations.*;

@Mapper     // 指定这是一个操作数据库的mapper，在编译之后会生成相应的接口实现类
public interface DepartmentMapper {

    @Select("select * from department where id=#{id}")
    public Department getDeptById(Integer id);

    @Delete("delete from department where id=#{id}")
    public int deleteDeptById(Integer id);

    @Options(useGeneratedKeys = true,keyProperty = "id")    //指明主键
    @Insert("insert into department(departmentName) values(#{departmentName})")    // 这里可以直接获取Department类中的参数
    public int insertDept(Department department);

    @Update("update department set departmentName=#{departmentName} where id=#{id}")
    public int updateDept(Department department);
}
```

接着编写测试文件进行测试：

```java
package com.dqj.springboot;

import com.dqj.springboot.bean.Department;
import com.dqj.springboot.mapper.DepartmentMapper;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class Springboot06DataMybatisApplicationTests {

    @Autowired
    DepartmentMapper departmentMapper;

    @Test
    public void testGetDeptById() {
        Department department = departmentMapper.getDeptById(1);
        System.out.println(department);
    }

    @Test
    public void testDeleteDeptById() {
        int res = departmentMapper.deleteDeptById(5);
        System.out.println(res);
    }

    @Test
    public void testInsertDept() {
        Department department = new Department(5,"管理部");
        int res = departmentMapper.insertDept(department);
        System.out.println(res);
    }

    @Test
    public void testUpdateDept() {
        Department department = new Department(6,"音乐部");
        int res = departmentMapper.updateDept(department);
        System.out.println(res);
    }
}
```

### 配置文件方式

配置文件方式和之前在 Spring 中使用的情况类似。

首先在`resources`文件夹下新建`mybatis-config.xml`和`mapper`文件夹，在`mapper`文件夹下新建`EmployeeMapper.xml`：
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/1.%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84.png)

`mybatis-config.xml`是 mybatis 的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 启用驼峰命名法 -->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
</configuration>
```

`EmployeeMapper.xml`是 mapper 文件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.dqj.springboot.mapper.EmployeeMapper">
    <select id="getEmpById" resultType="com.dqj.springboot.bean.Employee">
        SELECT * FROM employee WHERE id=#{id}
    </select>

    <insert id="insertEmp">
        INSERT INTO employee(lastName,email,gender,d_id) VALUES (#{lastName},#{email},#{gender},#{dId})
    </insert>
</mapper>
```

在`mapper`文件夹下新建`EmployeeMapper.java`接口类，注意方法名和上面的id一致：

```java
package com.dqj.springboot.mapper;

import com.dqj.springboot.bean.Employee;

// @Mapper或者@MapperScan将接口扫描装配到容器中，MapperScan写在入口类上面，并指明扫描的包
@Mapper
public interface EmployeeMapper {

    Employee getEmpById(Integer id);

    void insertEmp(Integer id);
}
```

接着在`application.yml`中指明**全局配置文件的位置**和**sql映射文件的位置**：

```yml
mybatis:
  config-location: classpath:mybatis/mybatis-config.xml   # 全局配置文件的位置
  mapper-locations: classpath:mybatis/mapper/*.xml        # 指定sql映射文件的位置
```

最后在配置类里进行测试即可：

```java
@Test
public void testGetEmpById() {
    Employee employee = employeeMapper.getEmpById(1);
    System.out.println(employee.getLastName());
}
```

## 使用SpringData JPA

SpringData 项目的目的是为了简化构建基于Spring 框架应用的数据访问技术，包括非关系数据库、Map-Reduce 框架、云数据服务等等；另外也包含对关系数据库的访问支持。
![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/2.SpringData.png)

首先引入相关依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

在`application.yml`中配置连接信息和 jpa：

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/jpa?serverTimezone=GMT
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
#      ddl-auto用来定义数据表的生成策略。update表示更新或者创建数据表结构
      ddl-auto: update
#    控制台显示sql
    show-sql: true
```

接着新建 entity 包，在该包下创建实体类，并添加注解（此时数据库中还没有表，这个实体类就可以用来进行表的创建）：

```java
package com.dqj.springboot.entity;

import javax.persistence.*;

/**
 * 这是一个普通的实体类
 * 使用JPA注解配置映射关系
 */
@Entity // 告诉JPA：这是一个实体类（和数据表映射的类）
@Table(name = "tbl_user")   // 指定和哪个数据表对应。如果省略，默认表名就是类名小写，即user。
public class User {

    @Id //这是一个主键
    @GeneratedValue(strategy = GenerationType.IDENTITY) //自增主键
    private Integer id;

    @Column(name = "last_name",length = 50) //指明和数据表对应的列
    private String lastName;
    @Column // 省略的话，则默认列名就是属性名
    private String email;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

接着创建 repository 包，在该包下创建 repository 接口。需要继承`JpaRepository`接口：

```java
package com.dqj.springboot.repository;

import com.dqj.springboot.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * 编写一个Dao接口来操作实体类对应的数据表
 * 继承JpaRepository来完成对数据库的操作
 * 这两个泛型中，第一个是操作的实体类，第二个泛型是实体类主键的类型
 */
public interface UserRepository extends JpaRepository<User,Integer> {
}
```

最后编写 controller 类进行测试：

```java
package com.dqj.springboot.controller;

import com.dqj.springboot.entity.User;
import com.dqj.springboot.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController {

    @Autowired
    UserRepository userRepository;

    @GetMapping("/user/{id}")
    public User getUser(@PathVariable("id") Integer id) {
        User one = userRepository.getOne(id);
        return one;
    }

    /**
     * 地址栏输入http://localhost:8080/user?lastName=zhangsan&email=zhangsan@qq.com即可完成添加
     * 添加的姓名和邮箱会组合成user对象的方式返回
     * @param user
     * @return
     */
    @GetMapping("/user")
    public User insertUser(User user) {
        User save = userRepository.save(user);
        return save;
    }
}
```

## 参考资料

1. [SpringData JPA的是用](https://www.jianshu.com/p/c23c82a8fcfc)