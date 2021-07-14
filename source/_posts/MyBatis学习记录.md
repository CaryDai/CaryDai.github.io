---
title: MyBatis学习记录
date: 2019-05-25 14:24:41
categories: Mybatis
tags: Mybatis
---

## MyBatis架构

![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3djsxc954j30x00s6421.jpg)

1. MyBatis配置
   SqlMapConfig.xml 是 MyBatis 的全局配置文件，配置了MyBatis 的运行环境等信息。
   Mapper.xml 文件是 sql 映射文件，文件中配置了操作数据库的语句。此文件需要在 SqlMapConfig.xml 中加载。
2. SqlSessionFactoryBuilder
   SqlSessionFactory 由 SqlSessionFactoryBuilder 创建，SqlSessionFactory 一旦创建完成就不需要 SqlSessionFactoryBuilder 了。
3. SqlSessionFactory
   SqlSessionFactory 即会话工厂，SqlSessionFactory 对象包含创建 SqlSession 实例的所有方法。它的最佳使用范围是整个应用运行期间，一旦创建后可以重复使用，通常以单例模式管理 SqlSessionFactory。
4. SqlSession
   操作数据库需要通过 SqlSession 进行，它是一个接口。
5. Executor
   MyBatis 底层自定义了 Executor 执行器接口操作数据库。
6. MappedStatement
   MappedStatement 也是 MyBatis 一个底层封装对象，它包装了 MyBatis 配置信息及 sql 映射信息等。Mapper.xml 中一个 sql 对应一个 MappedStatement 对象，sql id 即是 MappedStatement 的 id。
7. MappedStatement 对 sql 执行的输入参数进行定义，包括 HashMap、基本数据类型和pojo，Executor 通过 MappedStatement 在执行 sql 前将输入的 java 对象映射至 sql 中，输入参数映射就是 jdbc 编程中对 PreparedStatement 设置参数。
8. MappedStatement 对 sql 执行的输出结果进行定义，包括 HashMap、基本数据类型和pojo，Executor 通过 MappedStatement 在执行 sql 后将输出结果映射至 java 对象中，输出结果映射过程相当于 jdbc 编程中对结果的解析处理过程。

## Dao 开发方法

使用 Mybatis 开发 Dao，有两种方法：原始 Dao 开发方法和 Mapper 动态代理开发方法。

### 原始 Dao 开发方法

原始 Dao 开发方法需要程序员编写 Dao 接口和 Dao 实现类。

1. 编写pojo，User.java
   ```java
   package hdu.dqj.mybatis.pojo;
   import java.util.Date;
   
   public class User {
       private Integer id;
       private String username;// 用户姓名
       private String sex;// 性别
       private Date birthday;// 生日
       private String address;// 地址
   
       public Integer getId() {
           return id;
       }
   
       public void setId(Integer id) {
           this.id = id;
       }
   
       public String getUsername() {
           return username;
       }
   
       public void setUsername(String username) {
           this.username = username;
       }
   
       public String getSex() {
           return sex;
       }
   
       public void setSex(String sex) {
           this.sex = sex;
       }
   
       public Date getBirthday() {
           return birthday;
       }
   
       public void setBirthday(Date birthday) {
           this.birthday = birthday;
       }
   
       public String getAddress() {
           return address;
       }
   
       public void setAddress(String address) {
           this.address = address;
       }
   }
   ```
   
2. 编写映射文件user.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE mapper
           PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   <!--
        namespace：命名空间，用于隔离sql语句
        #{}：占位符，相当于jdbc的?
        ${}：字符串拼接指令，如果入参为普通数据类型，{}内只能写value
    -->
   <mapper namespace="user">
       <!--
           id：sql id，是sql语句的唯一标识；
           parameterType：入参的数据类型；
           resultType：返回结果的数据类型；
       -->
       <select id="getUserById" parameterType="int" resultType="hdu.dqj.mybatis.pojo.User">
           SELECT * from `user` where id = #{id}
       </select>
   
       <!-- resultType：如果返回结果为集合，只需设置为每一个元素的数据类型 -->
       <select id="getUserByName" parameterType="string" resultType="hdu.dqj.mybatis.pojo.User">
         	SELECT * from `user` where username like '%${value}%'
       </select>
   
       <!-- 插入用户 -->
       <!-- useGeneratedKeys：使用自增。 keyProperty与之配套使用，这里是user的主键 -->
       <insert id="insertUser" parameterType="hdu.dqj.mybatis.pojo.User" useGeneratedKeys="true" keyProperty="id">
         INSERT INTO USER (`username`, `birthday`, `sex`, `address`)
         VALUES (#{username}, #{birthday}, #{sex}, #{address});
       </insert>
   </mapper>
   ```

3. 编写 Dao 接口

   ```java
   package hdu.dqj.mybatis.dao;
   
   import hdu.dqj.mybatis.pojo.User;
   import java.util.List;
   
   /**
    * 用户信息持久化接口
    */
   public interface UserDao {
       /**
        * 根据用户ID查询用户信息
        * @param id
        * @return
        */
       User getUserById(Integer id);
   
       /**
        * 根据用户名查找用户列表
        * @param userName
        * @return
        */
       List<User> getUserByName(String userName);
   
       /**
        * 添加用户
        * @param user
        */
       void insertUser(User user);
   }
   ```

4. 编写实现类 UserDaoImpl.java （里面的SqlSessionFactoryUtils是抽取出来的工具类）
   工具类代码如下：

   ```java
   package hdu.dqj.mybatis.utils;
   
   import org.apache.ibatis.io.Resources;
   import org.apache.ibatis.session.SqlSessionFactory;
   import org.apache.ibatis.session.SqlSessionFactoryBuilder;
   
   import java.io.InputStream;
   
   /**
    * SqlSessionFactory工具类
    */
   public class SqlSessionFactoryUtils {
       private static SqlSessionFactory sqlSessionFactory;
       static {
           try {
               // 这个类会去加载核心配置文件 SqlMapConfig.xml
               SqlSessionFactoryBuilder ssfb = new SqlSessionFactoryBuilder();
               // 创建核心配置文件的输入流
               InputStream inputStream = Resources.getResourceAsStream("SqlMapConfig.xml");
               // 通过输入流创建SqlSessionFactory对象
               sqlSessionFactory = ssfb.build(inputStream);
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
   
       public static SqlSessionFactory getSqlSessionFactory() {
           return sqlSessionFactory;
       }
   }
   ```

   实现类代码如下：

   ```java
   package hdu.dqj.mybatis.dao.Impl;
   
   import hdu.dqj.mybatis.dao.UserDao;
   import hdu.dqj.mybatis.pojo.User;
   import hdu.dqj.mybatis.utils.SqlSessionFactoryUtils;
   import org.apache.ibatis.session.SqlSession;
   import org.apache.ibatis.session.SqlSessionFactory;
   
   import java.util.List;
   
   /**
    * 用户信息持久化实现
    */
   public class UserDaoImpl implements UserDao {
       @Override
       public User getUserById(Integer id) {
           SqlSessionFactory sqlSessionFactory = SqlSessionFactoryUtils.getSqlSessionFactory();
           SqlSession sqlSession = sqlSessionFactory.openSession();
           User user = sqlSession.selectOne("user.getUserById", id);
           sqlSession.close();
           return user;
       }
   
       @Override
       public List<User> getUserByName(String userName) {
           SqlSessionFactory sqlSessionFactory = SqlSessionFactoryUtils.getSqlSessionFactory();
           SqlSession sqlSession = sqlSessionFactory.openSession();
           List<User> list = sqlSession.selectList("user.getUserByName", userName);
           sqlSession.close();
           return list;
       }
   
       @Override
       public void insertUser(User user) {
           SqlSessionFactory sqlSessionFactory = SqlSessionFactoryUtils.getSqlSessionFactory();
           SqlSession sqlSession = sqlSessionFactory.openSession(true);
           sqlSession.insert("user.insertUser", user);
           sqlSession.close();
       }
   }
   ```

5. 配置 SqlMapConfig.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE configuration
           PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-config.dtd">
   <!-- dtd要求必须严格按照标签的顺序来写 -->
   <configuration>
       <properties resource="jdbc.properties" />
       <!-- 别名标签 -->
       <typeAliases>
           <!-- 单个别名定义，别名的使用不区分大小写 -->
           <typeAlias type="hdu.dqj.mybatis.pojo.User" alias="user" />
       </typeAliases>
       <!-- 环境集合属性对象，和spring整合后 environments配置将废除 -->
       <environments default="development">
           <!-- 环境子属性对象 -->
           <environment id="development">
               <!-- 使用jdbc事务管理 -->
               <transactionManager type="JDBC" />
               <!-- 数据库连接池 -->
               <dataSource type="POOLED">
                   <property name="driver" value="${jdbc.driver}" />
                   <property name="url" value="${jdbc.url}" />
                   <property name="username" value="${jdbc.username}" />
                   <property name="password" value="${jdbc.password}" />
               </dataSource>
           </environment>
       </environments>
   
       <!-- 加载映射文件 -->
       <mappers>
           <!-- 传统的DAO只有这种映射文件的加载方式 -->
           <mapper resource="hdu/dqj/mybatis/pojo/user.xml" />
       </mappers>
   </configuration>
   ```

6. 编写 JUnit 测试代码

   ```java
   package hdu.dqj.mybatis.dao.Impl;
   
   import hdu.dqj.mybatis.dao.UserDao;
   import hdu.dqj.mybatis.pojo.User;
   import org.junit.Test;
   
   import java.util.Date;
   import java.util.List;
   
   public class UserDaoImplTest {
   
       @Test
       public void getUserByIdTest() {
           UserDao userDao = new UserDaoImpl();
           User user = userDao.getUserById(27);
           System.out.println(user);
       }
   
       @Test
       public void getUserByNameTest() {
           UserDao userDao = new UserDaoImpl();
           List<User> list = userDao.getUserByName("张");
           for (User user : list) {
               System.out.println(user);
           }
       }
   
       @Test
       public void insertUserTest() {
           UserDao userDao = new UserDaoImpl();
           User user = new User();
           user.setUsername("张飞2");
           user.setSex("1");
           user.setBirthday(new Date());
           user.setAddress("杭电");
           userDao.insertUser(user);
       }
   }
   ```

### 存在的问题

- Dao 方法体存在重复代码：通过 SqlSessionFactory 创建 SqlSession，调用 SqlSession 的数据库操作方法。
- 调用 SqlSession 的数据库操作方法需要指定 statement 的 id，这里存在硬编码，不利于维护。

## Mapper 动态代理方法

Mapper 接口开发方法只需要程序员编写 Mapper 接口（相当于 Dao 接口），由 MyBatis 框架根据接口定义创建接口的动态代理对象，代理对象的方法体同上边 Dao 接口实现类方法。

Mapper 接口开发需要遵循以下四个规范：

1. Mapper.xml 文件中的 namespace 与 mapper 接口的类路径相同；
2. Mapper 接口的方法名和 Mapper.xml 文件中的每个 statement id 与 一致；
3. Mapper 接口方法的输入类型和 mapper.xml 中定义的每个 sql 的 parameterType 类型一致；
4. Mapper 接口方法的返回值 和 mapper.xml 中定义的每个 sql 的 resultType 类型一致。

### 开发步骤

1. 新建 mapper 文件夹，在该文件夹下新建 UserMapper.xml 文件

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE mapper
           PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   <!--
       DAO的动态代理开发规则：
       1、namespace必须是接口的全路径名
       2、接口的方法名必须与sql id一致
       3、接口的入参必须与parameterType类型一致
       4、接口的返回值必须与resultType类型一致
    -->
   <mapper namespace="hdu.dqj.mybatis.mapper.UserMapper">
       <select id="getUserById" parameterType="int" resultType="User">
           SELECT * from `user` where id = #{id}
       </select>
   
       <!-- resultType：如果返回结果为集合，只需设置为每一个的数据类型 -->
       <select id="getUserByName" parameterType="string" resultType="user">
         SELECT * from `user` where username like '%${value}%'
       </select>
   
       <!-- 插入用户 -->
       <!-- useGeneratedKeys：使用自增。 keyProperty与之配套使用，这里是user的主键 -->
       <insert id="insertUser" parameterType="hdu.dqj.mybatis.pojo.User" useGeneratedKeys="true" keyProperty="id">
         INSERT INTO USER
               (`username`,
                `birthday`,
                `sex`,
                `address`)
         VALUES (#{username},
                 #{birthday},
                 #{sex},
                 #{address});
       </insert>
   </mapper>
   ```

2. 编写接口文件 UserMapper.java

   ```java
   package hdu.dqj.mybatis.mapper;
   
   import hdu.dqj.mybatis.pojo.User;
   import java.util.List;
   
   /**
    * 用户信息持久化接口
    */
   public interface UserMapper {
       /**
        * 根据用户ID查询用户信息
        * @param id
        * @return
        */
       User getUserById(Integer id);
   
       /**
        * 根据用户名查找用户列表
        * @param userName
        * @return
        */
       List<User> getUserByName(String userName);
   
       /**
        * 添加用户
        * @param user
        */
       void insertUser(User user);
   }
   ```

3. 修改 SqlMapConfig.xml，加载新的映射文件：

   ```xml
   <!-- 加载映射文件 -->
   <mappers>
       <!-- 加载方式一，如果映射文件不在 mapper 目录下 -->   
       <!--<mapper resource="hdu/dqj/mybatis/pojo/UserMapper.xml" />-->
   
       <!--
               加载方式二，映射文件，class扫描器：
               1、接口文件必须与映射文件处于同一目录下
               2、接口文件的名称必须与映射文件的名称一致
           -->
       <!--<mapper class="hdu.dqj.mybatis.mapper.UserMapper" />-->
   
       <!--
               加载方式三，映射文件包扫描（推荐方式）：
               1、接口文件必须与映射文件处于同一目录下
               2、接口文件的名称必须与映射文件的名称一致
           -->
       <package name="hdu.dqj.mybatis.mapper" />
   </mappers>
   ```

4. 编写 JUnit 测试类

   ```java
   package hdu.dqj.mybatis.mapper;
   
   import hdu.dqj.mybatis.pojo.User;
   import hdu.dqj.mybatis.utils.SqlSessionFactoryUtils;
   import org.apache.ibatis.session.SqlSession;
   import org.junit.Test;
   
   import java.util.Date;
   import java.util.List;
   
   public class UserMapperTest {
   
       @Test
       public void getUserByIdTest() {
           SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
           // 获取接口的代理实现类
           UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
           User user = userMapper.getUserById(27);
           System.out.println(user);
           sqlSession.close();
       }
   
       @Test
       public void getUserByNameTest() {
           SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
           // 获取接口的代理实现类
           UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
           List<User> list = userMapper.getUserByName("张");
           for (User user : list) {
               System.out.println(user);
           }
           sqlSession.close();
       }
   
       @Test
       public void insertUserTest() {
           SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
           // 获取接口的代理实现类
           UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
           User user = new User();
           user.setUsername("刘备");
           user.setSex("1");
           user.setBirthday(new Date());
           user.setAddress("杭电");
           userMapper.insertUser(user);
           sqlSession.commit();
           sqlSession.close();
       }
   }
   ```

## 输入和输出映射

### 输入类型（parameterType）

上面的开发步骤演示了传递简单类型（使用 #{} 占位符或 ${} 进行 sql 拼接）和 pojo 对象（使用 ognl 表达式解析对象字段的值，#{} 或者 ${} 括号中的值为 pojo 属性名称）。

MyBatis 还可以传递 pojo 包装对象（pojo 类中的一个属性是另一个 pojo）。

需求：根据用户名模糊查询用户信息，查询条件放到 QueryVo 的 user 属性中。

1. 编写 QueryVo

   ```java
   package hdu.dqj.mybatis.pojo;
   
   import java.util.List;
   
   /**
    * 包装的pojo：对象里面还有对象
    */
   public class QueryVo {
       private User user;
   
       public User getUser() {
           return user;
       }
   
       public void setUser(User user) {
           this.user = user;
       }
   }
   ```

2. 编写 UserMapper.xml 文件

   ```xml
   <select id="getUserByQueryVo" parameterType="queryvo" resultType="user">
   	SELECT * from `user` where username like '%${user.username}%'
   </select>
   ```

3. 编写 UserMapper 接口

   ```java
   /**
   * 传递包装的pojo
   * @param vo
   * @return
   */
   List<User> getUserByQueryVo(QueryVo vo)
   ```

4. 编写测试方法

   ```java
   @Test
   public void testGetUserByQueryVo() {
       SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
       // 获取接口的代理实现类
       UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
       QueryVo vo = new QueryVo();
       User user = new User();
       user.setUsername("张");	// 查询名字里面有“张”字的人
       vo.setUser(user);
       List<User> list = userMapper.getUserByQueryVo(vo);
       for (User user1 : list) {
           System.out.println(user1);
       }
   }
   ```

### 输出类型（resultType）

可以输出简单类型（如：select count(*) from `user`），pojo 对象（如：SELECT * from `user` where id = #{id}），pojo列表（如：SELECT * from `user` where username like '%${value}%'）

### 使用 resultMap

resultType可以指定将查询结果映射为 pojo，但需要 pojo 的属性名和 sql 查询的列名一致方可映射成功。
如果 sql 查询字段名和 pojo 的属性名不一致，可以通过 resultMap 将字段名和属性名做一个对应关系。resultMap 实质上还需要将查询结果映射到 pojo 对象中。

需求：查询订单表 order 的所有数据，sql：select * from `order`

1. 编写 OrderMapper.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE mapper
           PUBLIC "-//hdu.dqj.mybatis.org//DTD Mapper 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   <mapper namespace="hdu.dqj.mybatis.mapper.OrderMapper">
       <!-- 使用resultMap：用于映射数据库和pojo之间的关系 -->
       <!-- type:映射成的pojo类型; id：resultMap唯一标识 -->
       <resultMap type="order" id="order_list_map">
           <!-- <id>用于映射主键 -->
           <id property="id" column="id" />
           <!-- 普通字段用<result>映射 -->
           <result property="userId" column="user_id" />
           <result property="number" column="number" />
           <result property="createtime" column="createtime" />
           <result property="note" column="note" />
       </resultMap>
       <select id="getOrderListMap" resultMap="order_list_map">
           select * from `order`
       </select>
   </mapper>
   ```

2. 编写接口 OrderMapper.java

   ```java
   List<Order> getOrderList();
   ```

3. 测试

   ```java
   @Test
   public void testGetOrderListMap() {
       SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
       OrderMapper orderMapper = sqlSession.getMapper(OrderMapper.class);
       List<Order> list = orderMapper.getOrderListMap();
       for (Order order : list) {
           System.out.println(order);
       }
       sqlSession.close();
   }
   ```

## 动态sql

MyBatis 提供了各种标签方法实现动态拼接 sql。

需求：根据性别和名字查询用户，sql：

```sql
select id, username, birthday, sex, address from `user` where sex = 1 and username like '%张%'
```

### sql 片段抽取

```sql
<sql id="user_sql">
	`id`,
    `username`,
    `birthday`,
    `sex`,
    `address`
</sql>
```

### where 标签和if 标签

```xml
<select id="getUserByPojo" parameterType="user" resultType="hdu.dqj.mybatis.pojo.User">
    select
    <!-- sql片段使用：refid：引用定义好的sql片段id -->
    <include refid="user_sql" />
    from `user`
    <!-- where标签自动补上where关键字，同时处理多余的and。用了where标签就不能再手动加上where关键字 -->
    <where>
        <if test="username != null and username != ''">
            and username like '%${username}%'
        </if>
        <if test="sex != null and sex != ''">
            and sex = #{sex}
        </if>
    </where>
</select>
```

### foreach标签

当向 sql 传递数组或 list 时，mybatis 使用 foreach 解析。

需求：根据多个 id 查询用户信息。sql：

```sql
select * from `user` where id in(1, 10, 16, 22, 24)
```

1. 向 UserMapper.xml 中添加 sql

   ```xml
   <select id="getUserByIds" parameterType="queryvo" resultType="user">
       select
       <include refid="user_sql" />
       from `user`
       <where>
           <!-- foreach循环标签。
                   collection：要遍历的集合，这里是QueryVo的ids属性
                   open: 循环开始之前输出的内容
                   item：设置循环的变量
                   separator：分隔符
                   close：循环结束之后输出的内容-->
           <foreach collection="ids" open="id in(" item="uId" separator="," close=")">
               #{uId}
           </foreach>
       </where>
   </select>
   ```

2. 在接口中添加查询方法：

   ```java
   /**
   * 演示foreach标签的使用，根据用户id列表查询用户
   * @param vo
   * @return
   */
   List<User> getUserByIds(QueryVo vo);
   ```

3. 测试

   ```java
   @Test
   public void testGetUserByIds() {
       SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
       UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
       QueryVo vo = new QueryVo();
       // 构建id列表
       vo.setIds(Arrays.asList(1,10,16,22,24));
       List<User> list = userMapper.getUserByIds(vo);
       for (User user : list) {
       	System.out.println(user);
       }
       sqlSession.close();
   }
   ```

   MyBatis把它解析为：

   ```sql
   select `id`, `username`, `birthday`, `sex`, `address` from `user` WHERE id in( ? , ? , ? , ? , ? ) 
   ```

## 关联查询

### 一对一查询

需求：查询所有订单信息，关联查询对应下单用户的信息。
sql 语句：

```sql
select o.`id`, o.`user_id` userId, o.`number`, o.`createtime`, o.`note`, u.username, u.address
from `order` o LEFT JOIN user u on u.id = o.user_id
```

- 使用 resultType
  新建 pojo 类 OrderUser，此类中包含了订单信息和用户信息，订单信息在 Order 类中，只需要继承它就行。再加上用户信息的字段即可：

  ```java
  public class OrderUser extends Order {
      private String username;
      private String address;
      ...	// get/set方法
  }
  ```

  OrderMapper.xml 中：

  ```xml
  <!-- 一对一关联查询：resultType的使用 -->
  <select id="getOrderUser" resultType="orderuser">
  	select
          o.`id`,
          o.`user_id` userId,
          o.`number`,
          o.`createtime`,
          o.`note`,
          u.username,
          u.address
  	from
          `order` o
          LEFT JOIN user u
          on u.id = o.user_id
  </select>
  ```

  Mapper 接口定义查询方法：

  ```java
  List<OrderUser> getOrderUser();
  ```

  测试方法：

  ```java
  @Test
  public void testGetOrderUserMap() {
      SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
      OrderMapper orderMapper = sqlSession.getMapper(OrderMapper.class);
      List<Order> list = orderMapper.getOrderUserMap();
      for (Order order : list) {
          System.out.println(order);
          System.out.println("此订单的用户为：" + order.getUser());
      }
      sqlSession.close();
  }
  ```

- 使用 resultMap

  可以继续使用上面的 OrderUser 类，此时 resultMap 写法如下：

  ```xml
  <!-- 一对一关联查询：resultMap的使用 -->
  <resultMap type="orderuser" id="order_user_map">
      <!-- <id>用于映射主键 -->
      <id property="id" column="id" />
      <!-- 普通字段用<result>映射，数据库的字段为user_id-->
      <result property="userId" column="user_id" />
      <result property="number" column="number" />
      <result property="createtime" column="createtime" />
      <result property="note" column="note" />
  
  </resultMap>
  <select id="getOrderUserMap" resultMap="order_user_map">
      select
          o.`id`,
          o.`user_id`,
          o.`number`,
          o.`createtime`,
          o.`note`,
          u.username,
          u.address
      from
          `order` o
          LEFT JOIN `user` u
          on u.id = o.user_id
  </select>
  ```

  添加接口：

  ```java
  List<OrderUser> getOrderUserMap();
  ```

  测试方法和上面 resultType 类似，只不过方法名需要改一下。

  显然这样写没有 resultType 方便。

  

  当然你也可以改造 pojo 类：在 Order 类中加入 User 属性，user 属性中用于存储关联查询的用户信息，因为订单关联查询用户是一对一关系，所以这里使用单个 User 对象存储关联查询的用户信息：

  ![](http://ww1.sinaimg.cn/large/005Mjp9lly1g3eqvziuyej30du042a9x.jpg)

  OrderMapper.xml ：

  ```xml
  <!-- 一对一关联查询：resultMap的使用 -->
  <resultMap type="order" id="order_user_map">
      <!-- <id>用于映射主键 -->
      <id property="id" column="id" />
      <!-- 普通字段用<result>映射，数据库的字段为user_id-->
      <result property="userId" column="user_id" />
      <result property="number" column="number" />
      <result property="createtime" column="createtime" />
      <result property="note" column="note" />
  
      <!-- association用于配置一对一关系，如果这个不配置，打印出来就为：user:null
               property：Order类里面的user属性
               javaType：user的数据类型，支持别名
          -->
      <association property="user" javaType="hdu.dqj.mybatis.pojo.User">
          <id property="id" column="user_id" />
          <result property="username" column="username" />
          <result property="address" column="address" />
          <result property="birthday" column="birthday" />
          <result property="sex" column="sex" />
      </association>
  </resultMap>
  <select id="getOrderUserMap" resultMap="order_user_map">
      select
          o.`id`,
          o.`user_id`,
          o.`number`,
          o.`createtime`,
          o.`note`,
          u.username,
          u.address,
          u.birthday,
          u.sex
      from
          `order` o
          LEFT JOIN `user` u
          on u.id = o.user_id
  </select>
  ```

  测试方法：

  ```java
  @Test
  public void testGetOrderUserMap() {
      SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
      OrderMapper orderMapper = sqlSession.getMapper(OrderMapper.class);
      List<Order> list = orderMapper.getOrderUserMap();
      for (Order order : list) {
          System.out.println(order);
          System.out.println("此订单的用户为：" + order.getUser());
      }
      sqlSession.close();
  }
  ```

### 一对多查询

需求：查询所有用户信息及用户关联的订单信息。

修改 pojo 类：在 User 类中加入 List<Order> orders 属性：

```java
private List<Order> orders
```

在 UserMapper.xml 中添加 sql：

```xml
<!-- type:映射成的pojo类型; id：resultMap唯一标识 -->
<resultMap type="user" id="user_order_map">
    <id property="id" column="id" />
    <result property="username" column="username" />
    <result property="address" column="address" />
    <result property="birthday" column="birthday" />
    <result property="sex" column="sex" />

    <!-- collection用于配置一对多关联
             property：User中的orders属性
             ofType：orders的数据类型，支持别名-->
    <collection property="orders" ofType="hdu.dqj.mybatis.pojo.Order">
        <!-- id标签用于绑定主键 -->
        <id property="id" column="oid"/>
        <!-- 使用result绑定普通字段 -->
        <result property="userId" column="id"/>
        <result property="number" column="number"/>
        <result property="createtime" column="createtime"/>
        <result property="note" column="note"/>
    </collection>
</resultMap>
<select id="getUserOrderMap" resultMap="user_order_map">
    select
        u.`id`,
        u.`username`,
        u.`birthday`,
        u.`sex`,
        u.`address`,
        u.`uuid2`,
        o.`id` oid,
        o.`number`,
        o.`createtime`,
        o.`note`
    from
        `user` u
        LEFT JOIN `order` o
        on o.user_id = u.id
</select>
```

UserMapper.java：

```java
/**
  * 演示一对多关联查询，resultMap
  * @return
  */
List<User> getUserOrderMap();
```

测试方法：

```java
@Test
public void testGetUserOrderMap() {
    SqlSession sqlSession = SqlSessionFactoryUtils.getSqlSessionFactory().openSession();
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    List<User> list = userMapper.getUserOrderMap();
    for (User user : list) {
        System.out.println(user);
        for (Order order : user.getOrders()) {
            if (order.getId() != null) {
                System.out.println("此用户下的订单有：" + order);
            }
        }
    }
    sqlSession.close();
}
```

