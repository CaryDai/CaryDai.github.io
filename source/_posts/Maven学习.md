---
title: Maven学习
date: 2021-06-30 10:34:37
tags: Maven
categories: 工具
description: 学习Maven知识
---

# Maven介绍

Apache Maven是一个项目管理和理解工具，它基于项目对象模型(project object model, POM)的概念，它可以管理项目的构建、报告和文档。

## 约定优于配置

|                目录                |                             目的                             |
| :--------------------------------: | :----------------------------------------------------------: |
|             ${basedir}             |                   存放pom.xml和所有子目录                    |
|      ${basedir}/src/main/java      |                       项目的java源代码                       |
|   ${basedir}/src/main/resources    |        项目的资源，比如说property文件，springmvc.xml         |
|      ${basedir}/src/test/java      |                项目的测试类，比如说Junit代码                 |
|   ${basedir}/src/test/resources    |                        测试类用的资源                        |
| ${basedir}/src/main/webapp/WEB-INF | web应用文件目录，web项目的信息，比如存放web.xml、本地图片、jsp视图页面 |
|         ${basedir}/target          |                打包输出目录（jar包就放在这）                 |
|     ${basedir}/target/classes      |                         编译输出目录                         |
|   ${basedir}/target/test-classes   |                       测试编译输出目录                       |
|             Test.java              |           Maven只会自动运行符合该命名规则的测试类            |
|          ~/.m2/repository          |                 Maven默认的本地仓库目录位置                  |

## Maven核心概念

在POM中，groupId、artifactId、packaging、version叫做maven坐标，它能唯一的确定一个项目。比较大的项目一般会分成几个子项目，在这种情况下，每个子项目就会有自己的POM文件，它们会有一个共同的父项目。这样只要构建父项目就能够构建所有的子项目了。同时子项目的POM会继承父项目的POM。

## Maven插件

Maven常用的插件比如`clean`、`compiler`、`deploy`、`jar`。为什么需要这些插件呢？因为maven本身不会做太多的事情，它不知道怎么样编译或者怎么样打包。它把构建的任务交给插件去做。插件定义了常用的构建逻辑，能够被重复利用。

## Maven生命周期

生命周期指项目的构建过程，它包含了一系列的有序的阶段，而一个阶段就是构建过程中的一个步骤，比如package阶段、compiler阶段等。那么生命周期阶段和上面说的插件目标之间是什么关系呢？**插件目标可以绑定到生命周期阶段上，一个生命周期可以绑定多个插件目标。当maven在构建过程中逐步的通过每个阶段时，会执行该阶段所有的插件目标。**目前生命周期阶段有clean、validate、compiler、test、package、verify、install、site、deploy阶段。

## Maven依赖管理

依赖关系是在dependencies部分中定义的。

```xml
<dependencies>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.11</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

`scope`决定了依赖关系的适用范围，比如`junit`的`scope`是`test`，那么它只会在执行`compiler:testCompile`和`surefire:test`目标的时候才会被加载到`classpath`中，在执行`compiler:compile`目标时是拿不到`junit`的。换句话说就是在`maven`哪个生命周期中起作用了。`scope`的默认值是`compile`，即任何时候都会被包含在`classpath`中，在打包的时候也会被包括进去。

## Maven多模块项目POM注意事项

### dependencyManagement

在项目开发过程中，有时一个项目下面包含了几个子模块，在多模块的情况，POM的配置应该要注意写什么呢？通过一个例子来说明下。
有这样一个工程，里面有A模块、B模块和C模块，A模块需要引入`junit`和`log4j`库，配置如下：

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
    <version>3.8.2</version>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
    <version>1.2.9</version>
</dependency>
```

此时B模块也需要引入这两个库，配置如下：

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactid>junit</artifactId>
    <version>4.8.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>log4j</groupId>
    <artifactid>log4j</artifactId>
    <version>1.2.16</version>
</dependency>
```

会发现A模块和B模块对`junit`和`log4j`库依赖的版本是不同的，出现这种情况是十分危险的，因为依赖不同版本的库可能会造成很多未知的风险。怎么解决不同模块之间对同一个库的依赖版本一样呢？Maven提供了优雅的解决办法，使用继承机制以及`dependencyManagement`元素来解决这个问题。如果你在父模块中配置`dependencies`，那么所有的子模块都自动继承，不仅达到了依赖一致的目的，还省了大段的代码，但这样来做会存在问题的。比如B模块需要`spring-aop`模块，但是C模块不需要`spring-aop`模块，如果用`dependencies`在父类中统一配置，C模块中也会包含有`spring-aop`模块，不符合我们的要求。但是用`dependencyManagement`就没有这样的问题。**dependencyManagement只会影响现有依赖的配置，但不会引入依赖。**这样我们在父模块中的配置可以更改如下所示：

```xml
<!-- dependencyManagement只会影响现有依赖的配置，但不会引入依赖。 -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>1.2.17</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```

这段配置不会给任何子模块引入依赖，如果某个子模块需要`junit`和`log4j`，只需要这样配置即可：

```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
    </dependency>
    <dependency>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
    </dependency>
</dependencies>
```

在多模块Maven项目中，使用`dependencyManagement`能够有效地帮我们维护依赖一致性。

> 注意：不要忘了在子模块中使用`parent`标签指定父模块，这样才能将依赖继承下来。

### pluginManagement

与`dependencyManagement`类似，我们可以使用`pluginManagement`元素管理插件。一个常见的用法就是我们希望项目所有模块使用`compiler`插件的时候，都是用`java1.8`，以及指定Java源文件编码为UTF-8，这时可以在父模块的POM中如下配置`pluginManagement`：

```xml
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>2.3.2</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
          <encoding>UTF-8</encoding>
        </configuration>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

这段配置会被应用到所有子模块的compiler插件中，因为Maven内置了compiler插件与生命周期的绑定，因此子模块不需要任何`maven-compiler-plugin`的配置了。

### properties

`properties`标签内可以把版本号作为变量进行声明，方便maven依赖标签用`${变量名}`的形式动态获取版本号。这样做的优点是当版本号发生改变时，仅仅需要更新`properties`标签中的变量值就行，不用更新所有依赖的版本号。

在多模块项目中，有时会看到根`pom.xml`文件中有`<properties>`标签，里面指定依赖的版本，比如：

```xml
<properties>
  <spring-core.version>5.3.8</spring-core.version>
</properties>
```

之后在依赖引入时就可以使用以下方式：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>${spring-core.version}</version>
</dependency>
```

### bom

通常公司会有一个团队专门维护一些通用的依赖版本，将这些版本信息固定到一个pom文件里，后期bom的维护方负责版本升级，确保各个依赖之间没有版本冲突。
本质上bom还是一个普通的pom文件，BOM是（Bill of Materials Dependency）依赖列表。一个bom例子如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.ydj.qd</groupId>
    <artifactId>inf-bom</artifactId>
    <version>1.0</version>
    <packaging>pom</packaging>

    <name>inf-bom</name>
    <description>第三方jar包统一管理</description>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring.version>4.3.15.RELEASE</spring.version>
    </properties>
  
    <dependencyManagement>
        <dependencies>
            <!-- 阿里 -->
            <!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>druid</artifactId>
                <version>1.1.12</version>
            </dependency>
            <!-- https://mvnrepository.com/artifact/com.aliyun.mns/aliyun-sdk-mns -->
            <dependency>
                <groupId>com.aliyun.mns</groupId>
                <artifactId>aliyun-sdk-mns</artifactId>
                <version>1.1.8</version>
                <classifier>jar-with-dependencies</classifier>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>fastjson</artifactId>
                <version>1.2.29</version>
            </dependency>

            <!-- Apache -->
            <dependency>
                <groupId>org.apache.commons</groupId>
                <artifactId>commons-lang3</artifactId>
                <version>3.3.2</version>
            </dependency>
            <dependency>
                <groupId>commons-collections</groupId>
                <artifactId>commons-collections</artifactId>
                <version>3.2.2</version>
            </dependency>
            <!-- https://mvnrepository.com/artifact/com.google.guava/guava -->
            <dependency>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
                <version>27.0.1-jre</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

业务方使用姿势：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.ydj.qd</groupId>
            <artifactId>inf-bom</artifactId>
            <version>1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </dependency>
    </dependencies>
</dependencyManagement>
```

用户只需要在`dependencyManagement`声明对`bom`的依赖，然后就可以在实际使用的地方去掉`version`字段就可以了, 如果需要使用不同于`bom`中的版本直接使用`<version>`覆盖即可。

## Maven常用的几个插件

### maven-compiler-plugin(编译插件)

用来编译Java代码，在对Java代码进行编译的时候，可以指定使用哪个JDK的版本来进行编译，配置如下所示：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.8.0</version>
  <configuration>
    <source>1.8</source> <!-- 源代码使用jdk1.8 -->
    <target>1.8</target> <!-- 使用jvm1.8编译目标代码 -->
  </configuration>
</plugin>
```

### maven-resources-plugin(资源插件)

Maven区别对待Java代码和资源文件，`maven-resources-plugin`则用来处理资源文件。默认的主资源文件目录是`src/main/resources`，很多时候会需要添加额外的资源文件目录，这个时候就可以通过配置`maven-resources-plugin`来实现，配置如下所示：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-resources-plugin</artifactId>
  <version>3.0.2</version>
  <executions>
    <execution>
      <phase>compile</phase> <!-- 与Maven编译生命周期绑定在一起 -->
    </execution>
  </executions>
</plugin>
```

### maven-surefire-plugin(测试插件)

Maven2/3中用于执行测试的插件不是`maven-test-plugin`，而是`maven-surefire-plugin`，其实在大部分情况下，只要你的测试类遵循通用的命令约定（以Test结尾，以TestCase结尾、或者Test开头），就几乎不用知晓该插件的存在。但是当你想要跳过测试、排除某些测试类、或者使用一些TestNG特性的时候，就要用到了`maven-surefire-plugin`的一些配置选项了，配置如下所示：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>2.22.1</version>
  <configuration>
    <skipTests>true</skipTests> <!-- 跳过测试 -->
  </configuration>
</plugin>
```

### maven-clean-plugin(清除插件)

主要作用就是清理构建目录下的全部内容，有些项目，构建时需要清理构建目录以外的文件，比如指定的库文件，这时候就需要配置`<filesets>`来实现了，配置如下所示：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-clean-plugin</artifactId>
  <version>3.1.0</version>
  <configuration>
    <!--<skip>true</skip>-->
    <!--<failOnError>false</failOnError>-->
    <!--当配置true时,只清理filesets里的文件,构建目录中得文件不被清理.默认是flase.-->
    <excludeDefaultDirectories>false</excludeDefaultDirectories>
    <filesets>
      <fileset>
        <!--要清理的目录位置-->
        <directory>${basedir}/logs</directory>
        <!--是否跟随符号链接 (symbolic links)-->
        <followSymlinks>false</followSymlinks>
      </fileset>
    </filesets>
  </configuration>
</plugin>
```

### maven-jar-plugin(打包插件)

生成当前项目的`jar`包（包含`.class`文件），以便以字节码形式分发给应用程序或库。一般没必要显示声明该插件，因为其绑定到了`maven`构建生命周期（Maven Build Life Cycle）。

```xml
<plugin>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.0.2</version>
</plugin>
```

### maven-source-plugin

生成当前项目的`jar`包（包含`java`源文件，即`.java`文件），以便允许IDE在调试时显示源代码，默认生成于`target`文件夹。其与包含`.class`的`jar`包联合使用。You can either bind this plugin to a phase or you can add it to a profile. https://maven.apache.org/plugins/maven-source-plugin/usage.html

```xml
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>3.2.0</version>
        <executions>
          <execution>
            <id>attach-sources</id>
            <phase>verify</phase>
            <goals>
              <goal>jar-no-fork</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ...
</project>
```

## 常用命令

### mvn clean

清理项目建的临时文件，一般是模块下的target目录。

### mvn package

打包到本项目，一般是在项目target目录下。（执行项目编译、单元测试、打包功能，单是不会布署到本地maven仓库和远程maven私服仓库）

### mvn install

打包会安装到本地的maven仓库中。（执行项目编译、单元测试、打包功能，同时布署到本地maven仓库，但不会布署到远程maven私服仓库）

### mvn deploy

将打包的文件发布到远程(如服务器)参考，提供其他人员进行下载依赖。（执行项目编译、单元测试、打包功能，同时会布署到本地maven仓库和远程maven私服仓库）

## Maven生命周期

Maven有三个内置的构建生命周期：default、clean和site。 default生命周期处理项目部署，clean生命周期处理项目清理，而site生命周期处理项目站点文档的创建。

默认生命周期包括以下几个阶段：validate、compile、test、package、verify、install、deploy。具体信息可以查看官方文档：https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html

## 总结

Maven是一个项目管理和自动化构建工具，项目遵循约定优于配置，这也是maven项目的一大特色。另外，maven本质上是一个插件框架，它的核心不执行任何具体的构建工作，全部都交给插件去执行，maven插件是与maven生命周期绑定在一起的。理解这些重要的核心点，对于maven的使用会有很大的帮助。

# 参考文档

- [Maven入门介绍](https://www.jianshu.com/p/39875424be3c)
- [Maven 中package、install、deploy的区别](https://www.jianshu.com/p/c6ccfbef7486)
- [Maven常用的四个命令](https://www.jianshu.com/p/1c02481ab82a)
- [maven bom的作用](https://www.jianshu.com/p/3a7f4d446d90)
- [Maven插件官方页面](https://maven.apache.org/plugins/index.html)
- [What's the difference between Maven Jar Plugin and Maven Source Plugin?](https://stackoverflow.com/questions/48137657/whats-the-difference-between-maven-jar-plugin-and-maven-source-plugin)

