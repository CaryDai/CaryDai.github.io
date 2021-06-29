---
title: Spring DI和AOP
date: 2019-09-16 20:36:25
categories: Spring
tags: Spring
description: 在看《Spring实战》的基础上总结 DI 和 AOP。
---

## 初探 DI 和 AOP

DI（依赖注入）和 AOP（面向切面编程）是 Spring 框架最核心的部分。（这里补充下 IOC 和 DI：IOC是控制反转的意思，在Spring中就是把类的实例的生成过程交给 Spring IOC容器来帮我们完成，不用我们手工创建；而 DI 是 IOC 的实现方法，通过依赖注入的方式告诉 IOC 容器：这里有个类，你帮我生成一下它的实例。IOC 容器可以容纳我们所开发的各种 bean）。

### DI

DI 的最大收益——**松耦合**。如果一个对象只是通过接口（而不是具体实现或初始化过程）来表明依赖关系，那么这种依赖就能够在对象毫不知情的情况下，用不同的具体实现进行替换。看书中的例子：

```java
public class BraveKnight implements Knight {
    private Quest quest;	// Quest 是任务接口
    public BraveKnight(Quest quest) {	// 通过传入接口的方式实现依赖注入
        this.quest = quest;
    }
    public void embarkOnQuest() {
        quest.embark();
    }
}
```

可以看到 BraveKnight 的构造函数中传入了一个 Quest 引用，这个 Quest 是所有任务的一个接口。你可能有许多特定的任务，但是我这个类通过传入任务接口的方式，不用去关心你具体实现的是哪个任务，我只需要通过调用接口的方法就能实现具体任务的功能。这其实是依赖注入的方式之一：构造器注入（constructor injection）。

现在我有一个“杀死一只怪龙”的任务：

```java
public class SlayDragonQuest implements Quest {
    private PrintStream stream;
    public SlayDragonQuest(PrintStream stream) {
        this.stream = stream;
    }
    public void embark() {
        stream.println("Embarking on quest to slay the dragon!");
	}
}
```

这样我这个具体的任务就可以传给 BraveKnight 类了。怎么给它呢？常见的是使用 XML 装配 bean（你可以把 bean 理解成类的代理或代言人）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="knight" class="com.dqj.spring.pojo.BraveKnight">
        <!-- 注入 Quest bean -->
        <constructor-arg ref="quest" />
    </bean>

    <bean id="quest" class="com.dqj.spring.pojo.SlayDragonQuest">
        <constructor-arg value="#{T(System).out}" />
    </bean>

</beans>
```

最后，Spring 通过应用上下文（Application Context）装载 bean 的定义并把它们组装起来。**Spring 应用上下文全权负责对象的创建和组装**。不同应用上下文的实现之间的主要区别在于如何加载配置，这里使用的是 ClassPathXmlApplicationContext 这个类的对象作为应用上下文：

```java
package com.dqj.spring.main;

import com.dqj.spring.interfaces.Knight;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * @author dqj
 * @date 2019/7/26/026
 */
public class KnightMain {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context =
                new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        Knight knight = (Knight)context.getBean("knight");
        knight.embarkOnQuest();		// 输出 Embarking on quest to slay the dragon!
    }
}
```

得到 Knight 对象的引用后，只需简单调用 embarkOnQuest() 方法就可以执行所赋予的探险任务了。**注意这个类完全不知道我们的英雄骑士接受哪种探险任务，而且完全没有意识到这是由 BraveKnight 来执行的。只有 applicationContext.xml 文件知道哪个骑士执行哪种任务。**

而如果用以前的方法你可能要这样写：

```java
public class OldMethod {

    @Test
    public void testOldMethod() {
        PrintStream stream = new PrintStream(System.out);
        SlayDragonQuest slayDragonQuest = new SlayDragonQuest(stream);
        Knight braveKnight = new BraveKnight(slayDragonQuest);
        braveKnight.embarkOnQuest();
    }
}
```

这样类与类之间的耦合度非常高，也就是说，如果我想换一个“打怪任务”或者换一个“骑士”，都需要在这里进行修改；而依赖注入的方式只需改变一下需要注入的类即可。如果工程大起来，DI 的优势会越来越明显。

### AOP

面向切面编程（aspect-oriented programming, AOP）允许你把遍布应用各处的功能分离出来形成可重用的组件。

一个系统内的各个模块可能都会需要诸如“日志”、“安全”这样的系统服务，如果这些服务的调用经常融入到模块自身的核心业务逻辑组件中，不仅修改起来麻烦（比如修改一个关注点（例如日志和服务）的逻辑，就需要在每个模块中进行修改），而且组件会因那些与自身核心业务无关的代码而变得混乱。

而 AOP 能使这些服务模块化，并以声明的方式将它们应用到它们需要影响的组件中去。所造成的结果就是这些组件会具有更高的内聚性并且会更加关注自身的业务。

看书中的例子：对于上面的骑士，我们假定在他执行任务前后都有位“诗人”为他吟诗，为此我们定义一个诗人类：

```java
public class Minstrel {
    private PrintStream stream;
    public Minstrel(PrintStream stream) {
        this.stream = stream;
    }
    // 骑士探险之前调用
    public void singBeforeQuest() {
        stream.println("Fa la la, the knight is so brave!");
    }
    // 骑士探险之后调用
    public void singAfterQuest() {
        stream.println("Tee hee hee, the brave knight did embark on a quest!");
    }
}
```

为了在 Spring 中使用切面，只需要在配置文件中进行相关配置即可：

```xml
...
<!-- 下面是在原来的基础上新增的配置 -->
<bean id="minstrel" class="com.dqj.spring.pojo.Minstrel">
    <constructor-arg value="#{T(System).out}" />
</bean>

<aop:config>
    <aop:aspect ref="minstrel">
        <!-- 定义切点 -->
        <aop:pointcut id="embark" expression="execution(* *.embarkOnQuest(..))" />
        <!-- 声明前置通知 -->
        <aop:before method="singBeforeQuest" pointcut-ref="embark" />
        <!-- 声明后置通知 -->
        <aop:after method="singAfterQuest" pointcut-ref="embark" />
    </aop:aspect>
</aop:config>
```

若执行上面的 main 方法，就会输出在“Embarking on quest to slay the dragon!”的前后各输出诗人的两句话。

我们可以从这个示例中获得**两个重要观点**：

1. 首先，Minstrel 仍然是一个 POJO，没有任何代码表明它要被作为一个切面使用。当我们在配置文件中配置后，在 Spring 的上下文中，Minstrel 实际上已经变成了一个切面了。
2. 其次，也是最重要的，Minstrel 可以被应用到 BraveKnight 中，而 BraveKnight 不需要显示地调用它。实际上，BraveKnight 完全不知道 Minstrel 的存在。

## 装配Bean

创建应用对象之间协作关系的行为通常称为装配（wiring），这也是依赖注入的本质。Spring 提供了三种装配 bean 的机制：

- 在 XML 中进行显式配置
- 在 Java 中进行显式配置
- 隐式的 bean 发现机制和自动装配

### 自动装配 bean

Spring 从两个角度来实现自动化装配：

- 组件扫描（component scanning）：Spring 会自动发现应用上下文中所创建的 bean。
- 自动装配（autowiring）：Spring 自动满足 bean 之间的依赖。

#### 组件扫描

根据书中CD播放器与CD的例子，CD播放器依赖于CD才能完成它的使命：

1. 创建 CompactDisc 接口，它定义了CD播放器对一盘CD所能进行的操作：

   ```java
   /**
    * Created by dqj on 2019/7/26/026 19:21
    * CD播放器接口
    */
   public interface CompactDisc {
       void play();
   }
   ```

2. 创建 CD 播放器的一个实现类：

   ```java
   @Component  // 表明该类会作为组件类，并告知 Spring 要为这个类创建 bean
   //@Component("mayday")	// 可以使用这种方式为这个 bean 设置 ID
   public class Autobiography implements CompactDisc {
       private String title = "If we never met";
       private String artist = "Mayday";
       public void play() {
           System.out.println("Playing " + title + " by " + artist);
       }
   }
   ```

   没有必要显式配置 Autobiographybean，因为这个类使用了`@Component`注解，所以Spring会为你把事情处理妥当。

3. 因为**组件扫描默认不启用**，所以需要显式配置一下 Spring，命令它去寻找带有`@Component`注解的类，并为其创建 bean。这里有两种方法，一种是以前一直在用的在配置文件中使用`<comtext:component-scan>`元素；另外一种是在与 Autobiography 类同一个包下，写一个类，这个类使用`@ComponentScan`注解：

   ```java
   @Configuration
   @ComponentScan      // 这个注解能够在 Spring 中启用组件扫描。
   public class CDPlayerConfig {
   }
   ```

   `@ComponentScan`默认会扫描与配置类相同的包（即与 CDPlayerConfig 类相同的包及其子包），查找带有`@Component`注解的类，并为其创建一个 bean。

4. 使用 JUnit4 进行测试：

   ```java
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration(classes = CDPlayerConfig.class)
   public class CDPlayerTest {
   
       @Autowired
       private CompactDisc cd;
   
       @Test
       public void cdShouldNotBeNull() {
           // 如果 cd 属性不为 null，说明 Spring 能够发现 CompactDisc 类，自动在 Spring 上下文中将其创建为 bean 并将其注入到测试代码中。
           assertNotNull(cd);
       }
   }
   ```

   在`assertNotNull(cd);`所在行打断点，进入 debug 模式，发现 cd 不为空，内容是：`cd: Autobiography@1528`。

#### 自动装配

简单来说，自动装配就是让 Spring 自动满足 bean 依赖的一种方法，在满足依赖的过程中，**会在 Spring 应用上下文中寻找匹配某个 bean 需求的其他 bean**。

下面的例子通过自动装配，将一个 CompactDisc 注入到 CDPlayer 中：

```java
@Component
public class CDPlayer implements MediaPlayer {

    private CompactDisc cd;

    // 表明当Spring创建CDPlayer bean的时候，会通过这个构造器来进行实例化并且会传入一个可设置给CompactDisc类型的bean
    @Autowired(required = false)
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public void player() {
        cd.play();
    }
}
```

如果没有匹配的 bean，那么在应用上下文创建的时候，Spring 会抛出一个异常。为了避免异常的出现，可以将`@Autowired`的`required`属性设置为 false。但是这一步需要谨慎对待，如果代码中没有进行 null 检查的话，这个处于未装配状态的属性有可能会出现`NullPointerException`。

### 通过Java代码装配bean

如果要将第三方库中的组件装配到你的应用中，在这种情况下，没有办法在它的类上添加`@Component`和`@Autowired`注解。此时，需要采用所谓的“显式配置”的方式。
有两种可选方案：Java 和 XML。作者认为 JavaConfig 是更好的方式，因为它更为强大、类型安全并且对重构友好。通常会将 JavaConfig 放到单独的包中，使它与其他的应用程序逻辑分离开来。

#### 创建配置类

首先我们需要创建 JavaConfig 类，关键在于为其添加`@Configuration`注解：

```java
@Configuration  //表明这个类是一个配置类，该类应该包含在Spring应用上下文中如何创建bean的细节。
public class CDPlayerConfig {
}
```

#### 声明简单的bean

要在 JavaConfig 中声明 bean，我们需要编写一个方法，这个方法会创建所需类型的实例，然后给这个方法添加`@Bean`注解。比如我添加一张五月天的《自传》CD：

```java
@Configuration
public class CDPlayerConfig {
    @Bean  //告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文中的bean。方法体中包含了最终产生bean实例的逻辑。
    public CompactDisc autoBiography() {
        return new Autobiography();
    }
}
```

默认情况下，bean 的 ID 与带有`@Bean`注解的方法名是一样的，这里 bean ID 就是 autoBiography。如果想设置成不同的ID，可以通过 name 属性指定：`@Bean(name = "...")`。

#### 借助JavaConfig实现注入

现在CD播放器需要插入CD才能播放音乐，所以 CDPlayerbean 依赖于 CompactDiscbean。
在 JavaConfig 中装配 bean 的最简单方式就是引用创建 bean 的方法（这里指的方法是 autoBiography），因为该方法会返回需要被装配的 bean 的类型（这里就是 CompactDisc 类型）：

```java
@Bean
public CDPlayer cdPlayer() {
    return new CDPlayer(autoBiography());
}
```

因为 autoBiography() 方法上添加了`@Bean`注解，Spring将会拦截所有对它的调用，并确保直接返回该方法所创建的bean，而不是每次都对其进行实际的调用。Spring 中的 bean 都是单例的。

通过调用方法来引用 bean 的方式有点令人困惑，可以使用下面这种方式：

```java
@Bean
public CDPlayer cdPlayer2(CompactDisc compactDisc) {
    return new CDPlayer(compactDisc);
}
```

通过这种方式引用其他的bean通常是最佳的选择，因为它不会要求将 CompactDisc 声明到同一个配置类之中。在这里甚至没有要求 CompactDisc 必须要在 JavaConfig 中声明，实际上它可以通过组件扫描功能自动发现或者通过XML来进行配置。

最后可以自己写个 main 函数来进行测试：

```java
package com.dqj.springbean2.main;

import com.dqj.springbean2.Configuration.CDPlayerConfig;
import com.dqj.springbean2.pojo.CDPlayer;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class ConfigMain {
    public static void main(String[] args) {

        // 通过Java配置来实例化Spring容器
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(CDPlayerConfig.class);

        // 从Spring容器中获取Bean对象
        CDPlayer cdPlayer2 = (CDPlayer)context.getBean("cdPlayer2");

        // 调用对象中的方法
        cdPlayer2.player();
    }
}
```

会输出：`Playing If we never met by Mayday`。

### 通过XML配置bean

这个一直在用，就不记录了。

### 导入和混合配置

#### 在JavaConfig中引用XML配置

现在假设需要对 CDPlayerConfig 进行拆分，虽然它只定义了两个 bean。先来看一下之前使用 JavaConfig 装配 bean 方式中的 CDPlayerConfig：

```java
@Configuration  //表明这个类是一个配置类，该类应该包含在Spring应用上下文中如何创建bean的细节。
public class CDPlayerConfig {

    @Bean  //告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文中的bean。方法体中包含了最终产生bean实例的逻辑。
    public CompactDisc autoBiography() {
        return new Autobiography();
    }
    
    /**
     * 理解起来更为简单的方式
     * @param compactDisc
     * @return
     */
    @Bean
    public CDPlayer cdPlayer(CompactDisc compactDisc) {
        return new CDPlayer(compactDisc);
    }
}
```

1. 如果不使用 XML 配置：
   将那个简单 bean 抽取出来（这里的简单 bean 就是 CompactDiscbean），定义到它自己的 CDConfig 类中：

   ```java
   @Configuration
   public class CDConfig {
       @Bean
       public CompactDisc compactDisc() {
           return new Autobiography();
       }
   }
   ```

   接着需要将两个类（CDConfig 和 CDPlayerConfig）组合起来。一种方法是在 CDPlayerConfig 中使用`@Import`注解导入 CDConfig：

   ```java
   @Configuration
   @Import(CDConfig.class)
   public class CDPlayerConfig {
   	@Bean
       public CDPlayer cdPlayer(CompactDisc compactDisc) {
           return new CDPlayer(compactDisc);
       }
   }
   ```

   或者采用一个更好的办法，也就是不在 CDPlayerConfig 中使用`@Import`，而是创建一个更高级别的SoundSystemConfig，在这个类中使用`@Import`将两个配置类组合在一起：

   ```java
   @Configuration
   @Import(CDPlayerConfig.class, CDConfig.class)
   public class SoundSystemConfig{
   }
   ```

2. 通过 XML 配置

   ```xml
   <bean id="compactDisc" class="soundsystem.BlankDisc">
       <constructor-arg>
           <list>
               <value>If we never met</value>
               <value>Good good</value>
               <value>What's your story</value>
           </list>
       </constructor-arg>
   </bean>
   ```

   可以使用`@ImportResource`注解让 Spring 同时加载它和其他基于 Java 的配置。假设 CompactDisc 定义在名为 `cd-config.xml`的文件中，该文件位于根类路径下，那么可以修改 SoundSystemConfig：

   ```java
   @Configuration
   @Import(CDPlayerConfig.class)
   @ImportResource("classpath:cd-config.xml")
   public class SoundSystemConfig{
   }
   ```

#### 在XML配置中引用JavaConfig

如果有越来越多的 bean 在 XML 中配置，现在假设不再将 CompactDisc 配置在 XML 中，而是将其配置在 JavaConfig 中，CDPlayer 则继续配置在 XML 中。此时需要在 XML 中引入 JavaConfig 的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="soundsystem.CDConfig" />
    
    <bean id="cdPlayer" class="soundsystem.CDPlayer" c:cd-ref="compactDisc" />
</beans>
```

类似地，你可能还希望创建一个更高层次的配置文件，这个文件不声明任何的 bean，只是负责将两个或更多的配置组合起来：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="soundsystem.CDConfig" />
    
    <import resource="cdplayer-config.xml" />
</beans>
```

## 高级装配

### 环境与profile

生产环境和开发环境会用到不同的 bean 定义，在3.1版本中，Spring 引入了 bean profile 的功能。要使用 profile，你首先要将所有不同的bean定义整理到一个或多个 profile 之中，在将应用部署到每个环境时，要确保对应的 profile 处于激活（active）的状态。

在Java配置中，可以使用**@Profile**注解指定某个 bean 属于哪一个 profile：

```java
@Configuration
public class DataSourceConfig {
    
    @Bean(destroyMethod = "shutdown")
    @Profile("dev")	//它会告诉 Spring 这个配置类中的bean只有在dev profile激活时才会创建。如果dev profile 没有激活的话，那么带有 @Bean 注解的方法都会被忽略掉。
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            ....
    }
    
    @Bean
    @Profile("prod")
    public DataSource ...() {
        ...
    }
}
```

激活profile：Spring在确定哪个profile处于激活状态时，需要依赖两个独立的属性：spring.profiles.active和spring.profiles.default。如果设置了spring.profiles.active属性的话，那么它的值就会用来确定哪个profile是激活的。但如果没有设置spring.profiles.active属性的话，那Spring将会查找spring.profiles.default的值。如果spring.profiles.active和spring.profiles.default均没有设置的话，那就没有激活的profile，因此只会创建那些没有定义在profile中的bean。激活的方式可以看书52/53页。

### 条件化的bean

Spring 4引入了一个新的**@Conditional**注解，它可以用到带有**@Bean**注解的方法上。如果给定的条件计算结果为 true，就会创建这个 bean，否则的话，这个 bean 会被忽略。

### 处理自动装配的歧义性

假设我们使用**@Autowired**注解标注了 setDessert() 方法：

```java
@Autowired
public void setDessert(Dessert dessert) {
    this.dessert = dessert;
}
```

在本例中，Dessert 是一个接口，并且有三个类实现了这个接口，分别为 Cake、Cookies 和 IceCream：

```java
@Component
public class Cake implements Dessert {...}

@Component
public class Cookies implements Dessert {...}

@Component
public class IceCream implements Dessert {...}
```

因为这三个实现均使用了**@Component**注解，在组件扫描的时候，能够发现它们并将其创建为 Spring 应用上下文里面的bean。然后，当 Spring 试图自动装配 setDessert() 中的 Dessert 参数时，它并没有唯一、无歧义的可选值。

解决方法：

- 将可选 bean 中的某一个设为首选（primary）的 bean。

  - 用在组件扫描的 bean 上：

    ```java
    @Component
    @Primary
    public class IceCream implements Dessert {...}
    ```

  - 通过 Java 配置显式地声明 IceCream：

    ```java
    @Bean
    @Primary
    public Dessert iceCream() {
        return new IceCream();
    }
    ```

  - 使用 XML 方式配置 bean 时：

    ```xml
    <bean id="iceCream" class="com.desserteater.IceCream" primary="true" />
    ```

- 使用限定符（qualifier）来帮助 Spring 将可选的 bean 的范围缩小到只有一个 bean。
  **@Qualifier**注解是使用限定符的主要方式。它可以与**@Autowired**和**@Inject**协同使用，在注入的时候指定想要注入进去的是哪个bean。
  例如，我们想要确保将 IceCream 注入到 setDessert() 之中：

  ```java
  @Autowired
  @Qualifier("iceCream")	// 为@Qualifier注解所设置的参数就是想要注入的bean的ID。
  public void setDessert(Dessert dessert) {
      this.dessert = dessert;
  }
  ```

  所有使用**@Component**注解声明的类都会创建为 bean，并且 bean 的 ID 为首字母变为小写的类名。因此，**@Qualifier("iceCream")**指向的是组件扫描时所创建的 bean，并且这个 bean 是 IceCream 类的实例。

  有个问题，如果 bean 的名字改变的话，bean ID 也会随之变化，这时可以创建自定义的限定符。这里所需要做的就是在 bean 声明上添加**@Qualifier**注解：

  ```java
  @Component
  @Qualifier("cold")
  public class IceCream implements Dessert {...}
  ```

  在这种情况下，cold 限定符分配给了 IceCreambean。因为它没有耦合类名，因此你可以随意重构 IceCream 的类名，而不必担心会破坏自动装配。在注入的地方，只要引用 cold 限定符就可以了：

  ```java
  @Autowired
  @Qualifier("cold")
  public void setDessert(Dessert dessert) {
      this.dessert = dessert;
  }
  ```

  你甚至可以自定义限定符注解，详细可以看书本 57 页。

### bean的作用域

默认情况下，Spring 应用上下文中所有 bean 都是作为以单例（singleton）的形式创建的。也就是说，不管给定的一个bean 被注入到其他 bean 多少次，每次所注入的都是同一个实例。

Spring定义了多种作用域，可以基于这些作用域创建bean，包括：

- 单例（Singleton）：在整个应用中，只创建bean的一个实例。
- 原型（Prototype）：每次注入或者通过Spring应用上下文获取的时候，都会创建一个新的bean实例。
- 会话（Session）：在Web应用中，为每个会话创建一个bean实例。
- 请求（Rquest）：在Web应用中，为每个请求创建一个bean实例。

如果选择其他的作用域，要使用**@Scope**注解，它可以与**@Component**或**@Bean**一起使用。例如：

- 使用组件扫描的方式：

    ```java
    @Component
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public class Notepad {...}
    ```

    这里，使用 ConfigurableBeanFactory 类的 SCOPE_PROTOTYPE 常量设置了原型作用域。你当然也可以使用@Scope("prototype")，但是使用 SCOPE_PROTOTYPE 常量更加安全并且不易出错。

- Java 配置方式：

    ```java
    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public Notepad notepad() {
    	return new Notepad();
    }
    ```

- XML 配置方式：

    ```xml
    <bean id="notepad" class="com.myapp.Notepad" scope="prototype" />
    ```

#### 使用会话和请求作用域

就购物车bean来说，会话作用域是最为合适的，因为它与给定的用户关联性最大：

```java
@Component
@Scope(value=WebApplicationContext.SCOPE_SESSION,
      proxyMode=ScopedProxyMode.INTERFACES)
public ShoppingCart cart() {...}
```

这里，我们将 value 设置成了 WebApplicationContext 中的 SCOPE_SESSION 常量（它的值是session）。这会告诉Spring 为 Web 应用中的每个会话创建一个 ShoppingCart。这会创建多个 ShoppingCart bean 的实例，**但是对于给定的会话只会创建一个实例**，在当前会话相关的操作中，这个 bean 实际上相当于单例的。

**@Scope**同时还有一个 proxyMode 属性，它被设置成了 ScopedProxyMode.INTERFACES。这个属性解决了将会话或请求作用域的 bean 注入到单例 bean 中所遇到的问题。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/proxyMode.png)

#### 在XML中声明作用域代理

```xml
<bean id="cart" class="com.myapp.ShoppingCart" scope="session">
	<aop:scope-proxy />
</bean>
```

`<aop:scoped-proxy>`是与**@Scope**注解的 proxyMode 属性功能相同的 Spring XML 配置元素。它会告诉 Spring 为 bean创建一个作用域代理。默认情况下，它会使用**CGLib**创建目标类的代理。但是我们也可以将 proxy-target-class 属性设置为 false，进而要求它生成基于接口的代理：

```xml
<bean id="cart" class="com.myapp.ShoppingCart" scope="session">
	<aop:scope-proxy proxy-target-class="false" />
</bean>
```

### 运行时值注入

Spring 提供了两种在运行时求值的方式：

- 属性占位符（Property placeholder）。
- Spring 表达式语言（SpEL）。

#### 注入外部的值

在Spring中，处理外部值的最简单方式就是声明属性源并通过Spring的 Environment 来检索属性。例如：

```java
@Configuration
@PropertySource("classpath:/com/soundsystem/app.properties")	// 声明属性源
public class ExpressiveConfig {
    
    @Autowired
    Environment env;
    
    @Bean
    public BlankDisc disc() {
		// 检索属性值
		return new BlankDisc(env.getProperty("disc.title"), env.getProperty("disc.artist"));
    }
}
```

在本例中，**@PropertySource**引用了类路径中一个名为 app.properties 的文件。它大致会如下所示：

```properties
disc.title=Autobiography
disc.artist=Mayday
```

这个属性文件会加载到Spring的 Environment 中，稍后可以从这里检索属性。同时，在 disc() 方法中，会创建一个新的BlankDisc，它的构造器参数是从属性文件中获取的，而这是通过调用 getProperty() 实现的。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/2.%E6%B7%B1%E5%85%A5%E5%AD%A6%E4%B9%A0Spring%E7%9A%84Environment.png)

##### 解析属性占位符

在Spring装配中，占位符的形式为使用“${... }”包装的属性名称。作为样例，我们可以在XML中按照如下的方式解析BlankDisc 构造器参数：

```xml
<bean id="sgtPeppers" class="SOUNDsystem.BlankDisc"
      c:_title="${disc.title}"
      c:_artist="${disc.artist}"/>
```

可以看到，title 构造器参数所给定的值是从一个属性中解析得到的，这个属性的名称为 disc.title。artist 参数装配的是名为
disc.artist 的属性值。按照这种方式，XML配置没有使用任何硬编码的值，它的值是从配置文件以外的一个源中解析得到的。

#### 使用Spring表达式语言进行装配

SpEL 能够以一种强大和简洁的方式将值装配到 bean 属性和构造器参数中，在这个过程中所使用的表达式会在运行时计算得到值。

SpEL 表达式要放在“#{...}”之中，这与属性占位符有些类似，属性占位符需要放到“${...}”之中。

如果通过组件扫描创建bean的话，在注入属性和构造器参数时，我们可以使用**@Value**注解：

```java
public BlankDisc(@Value("#{systemProperties['disc.title']}") String title, 
                 @Value("#{systemProperties['disc.artist']}") String artist) {
    this.title = title;
    this.artist = artist;
}
```

在XML配置中，你可以将SpEL表达式传入<property>或<constructor-arg>的value属性中，或者将其作为**p-命名空间**或**c-命名空间**条目的值。例如，在如下BlankDisc bean的XML声明中，构造器参数就是通过SpEL表达式设置的：

```xml
<bean id="sgtPeppers" class="SOUNDsystem.BlankDisc"
      c:_title="#{systemProperties['disc.title']}"
      c:_artist="#{systemProperties['disc.artist']}"/>
```

注：在通过构造方法或`set`方法给`bean`注入关联项时通常是通过`constructor-arg`元素和`property`元素来定义的。在有了`p`命名空间和`c`命名空间时我们可以简单的把它们当做`bean`的一个属性来进行定义。

详细的SpEL表达式的用法可以看书。

## 面向切面的Spring

AOP中的术语：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/AOP%E7%9A%84%E7%9B%B8%E5%85%B3%E6%9C%AF%E8%AF%AD.png)

### Spring对AOP的支持

Spring提供了4种类型的AOP支持：

- 基于代理的经典Spring AOP；
- 纯POJO切面；
- **@AspectJ**注解驱动的切面；
- 注入式AspectJ切面（适用于Spring各版本）。

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/3.Spring%E5%9C%A8%E8%BF%90%E8%A1%8C%E6%97%B6%E9%80%9A%E7%9F%A5%E5%AF%B9%E8%B1%A1.png)

## 参考资料

1. 《Spring实战4.0》
2. [spring IoC的概念](https://www.cnblogs.com/ooo0/p/10962330.html)