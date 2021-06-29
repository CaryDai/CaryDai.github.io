---
title: Java动态代理
date: 2019-12-10 21:25:19
tags:
	- 代理模式
	- JDK动态代理
	- Cglib动态代理
categories: Java
description:
---

介绍这个知识点的博客已经有很多了，我把自己不太熟悉的点记录下，因为我发现看了以后不记下来或者不经常用，某个知识点很容易忘记。

## 代理模式

代理模式是一种设计模式，提供了对目标对象额外的访问方式，即通过代理对象访问目标对象，这样可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。

简言之，代理模式就是设置一个中间代理来控制访问原目标对象，以达到增强原对象的功能和简化访问方式。

代理模式UML类图:

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/1.%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8FUML%E5%9B%BE.png)

- Subject: 定义代理类和 RealSubject 的公共对外方法，也是代理类代理 RealSubject 的方法；
- RealSubject: 真正实现业务逻辑的类;
- Proxy: 代理类.

如果**根据字节码的创建时机**来分类，可以分为静态代理和动态代理：

- 所谓**静态**也就是在**程序运行前**就已经存在代理类的**字节码文件**，代理类和真实主题角色的关系在运行前就确定了。
- 而动态代理的源码是在程序运行期间由**JVM**根据反射等机制**动态的生成**，所以在运行前并不存在代理类的字节码文件。

## 静态代理

静态代理的例子可以直接看[这篇博客](https://juejin.im/post/5c1ca8df6fb9a049b347f55c). 例子很简单, 通过静态代理,  达到了功能增强的目的，而且没有侵入原代码，这是静态代理的一个优点。

### 静态代理的缺点

虽然静态代理实现简单，且不侵入原代码，但是，当场景稍微复杂一些的时候，静态代理的缺点也会暴露出来。

1、 当需要代理多个类的时候，由于代理对象要实现与目标对象一致的接口，有两种方式：

- 只维护一个代理类，由这个代理类实现多个接口，但是这样就导致**代理类过于庞大**
- 新建多个代理类，每个目标对象对应一个代理类，但是这样会**产生过多的代理类**

2、 当接口需要增加、删除、修改方法的时候，目标对象与代理类都要同时修改，**不易维护**。

### 如何改进?

**让代理类动态地生成**。需要用到某个类的代理类时，再生成对应的代理类，即运行时生成类。

==为什么类可以动态生成?==

这个问题得回到类是怎么加载的？由《深入理解Java虚拟机》中可以知道类加载过程分为以下几步：

加载、验证、准备、解析、初始化。其中加载阶段完成以下3件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在内存中生成一个代表这个类的 `java.lang.Class` 对象，作为方法区这个类的各种数据访问入口。

由于虚拟机规范对这3点要求并不具体，所以实际的实现是非常灵活的，关于第1点，**获取类的二进制字节流**（class字节码）就有很多途径，有一条是：“**运行时计算生成**，这种场景使用最多的是动态代理技术，在 java.lang.reflect.Proxy 类中，就是用了 ProxyGenerator.generateProxyClass 来为特定接口生成形式为 `*$Proxy` 的代理类的二进制字节流。”

所以现在需要想办法，根据接口或目标对象，得到代理类的字节码，然后再加载到 JVM 中使用。

### 常见的字节码操作类库

- Apache BCEL (Byte Code Engineering Library)：是Java classworking广泛使用的一种框架，它可以深入到JVM汇编语言进行类操作的细节。
- ObjectWeb ASM：是一个Java字节码操作框架。它可以用于直接以二进制形式动态生成stub根类或其他代理类，或者在加载时动态修改类。
- CGLIB(Code Generation Library)：是一个功能强大，高性能和高质量的代码生成库，用于扩展JAVA类并在运行时实现接口。
- Javassist：是Java的加载时反射系统，它是一个用于在Java中编辑字节码的类库; 它使Java程序能够在运行时定义新类，并在JVM加载之前修改类文件。
- ...

## 动态代理

静态代理与动态代理的区别主要在：

- 静态代理在编译时就已经实现，编译完成后代理类是一个实际的class文件。
- 动态代理是在运行时动态生成的，即编译完成后没有实际的class文件，而是在运行时动态生成类字节码，并加载到JVM中。

### JDK动态代理

动态地在内存中构建代理对象，从而实现对目标对象的代理功能。JDK动态代理又被称为“接口代理”。

JDK动态代理对象不需要实现接口，但是要求目标对象必须实现接口。

JDK动态代理主要涉及的类有：

- `java.lang.reflect.Proxy`，主要方法为：

  ```java
  //返回一个指定接口的代理类实例，该接口可以将方法调用指派到指定的调用处理程序。
  static Object newProxyInstance(ClassLoader loader,  //指定当前目标对象使用类加载器
   Class<?>[] interfaces,    //目标对象实现的接口的类型
   InvocationHandler h      //事件处理器
  )
  ```

- `java.lang.reflect.InvocationHandler`，主要方法为：

  ```java
  // 在代理实例上处理方法调用并返回结果。
  Object invoke(Object proxy, Method method, Object[] args)
  ```

下面看一个例子（代码来自[这篇博客](https://segmentfault.com/a/1190000011291179#item-1)）：

假设现在有一个对外提供的接口`IUserDao`：

```java
// 对外提供的接口
public interface IUserDao {
    public void save();
}
```

它的实现类`UserDao`如下：

```java
/**
 * 目标类，或者说真实的业务逻辑类。等下这个类会被代理。
 */
public class UserDao implements IUserDao {
    @Override
    public void save() {
        System.out.println("保存数据");
    }
}
```

现在为上面的目标类编写一个JDK动态代理类`ProxyFactory`：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * JDK动态代理类
 */
public class ProxyFactory {
    private Object target;  // 维护一个目标对象

    public ProxyFactory(Object target) {
        this.target = target;
    }

    // 为目标对象生成代理对象的方法
    public Object getProxyInstance() {
        // 返回一个指定接口的代理类实例
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), 
            new InvocationHandler(){
            
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("开启事务");
                    // 执行目标对象target的method方法，参数为args
                    Object resultValue = method.invoke(target, args);
                    System.out.println("提交事务");
                    return resultValue;
                }
            });
    }
}
```

最后编写测试类：

```java
/**
 * 测试类
 */
public class TestProxy {
    public static void main(String[] args) {
        IUserDao target = new UserDao();
        System.out.println(target.getClass());  //输出目标对象信息
        IUserDao proxy = (IUserDao) new ProxyFactory(target).getProxyInstance();    // 得到代理对象
        System.out.println(proxy.getClass());  //输出代理对象信息
        proxy.save();  //执行代理方法
    }
}
```

输出：

```
class UserDao
class com.sun.proxy.$Proxy0
开启事务
保存数据
提交事务
```

这里一开始我对最后的输出有疑惑，`proxy.save()`是怎么输出最后三句话的？看样子只能跟`InvocationHandler`有关了。

由[这篇博客](https://blog.csdn.net/yaomingyang/article/details/80981004)可以知道，“InvocationHandler接口是proxy代理实例的调用处理程序实现的一个接口，每一个proxy代理实例都有一个关联的调用处理程序；在代理实例调用方法时，方法调用被编码分派到调用处理程序的invoke方法。”也就是说，**测试类的最后一句`proxy.save();`中，当代理实例（proxy）调用 save() 方法时，会执行`new InvocationHandler(){...}`中的 invoke() 方法。**

每一个动态代理类的调用处理程序都必须实现 InvocationHandler 接口，并且每个代理类的实例都关联到了实现该接口的动态代理类调用处理程序中，当我们通过动态代理对象调用一个方法时候，这个方法的调用就会被转发到实现InvocationHandler 接口类的 invoke 方法来调用。

这下清楚了。

### Cglib动态代理

cglib (Code Generation Library )是一个第三方代码生成类库，==运行时在内存中动态生成一个子类对象从而实现对目标对象功能的扩展==。

#### cglib特点

- JDK的动态代理有一个限制，就是使用JDK动态代理的对象必须实现一个或多个接口。如果想代理没有实现接口的类，就可以使用CGLIB实现。
- CGLIB是一个强大的高性能的代码生成包，它可以在运行期扩展Java类与实现Java接口。它广泛的被许多AOP的框架使用，例如Spring AOP和dynaop，为他们提供方法的interception（拦截）。
- CGLIB包的底层是通过使用一个小而快的字节码处理框架ASM，来转换字节码并生成新的类。不鼓励直接使用ASM，因为它需要你对JVM内部结构包括class文件的格式和指令集都很熟悉。

使用 cglib 需要引入 cglib 的 jar 包。

下面看一个例子（代码同样来自[这篇博客](https://segmentfault.com/a/1190000011291179#item-1)）：

目标对象：`UserDao`：

```java
public class UserDao{
    public void save() {
        System.out.println("保存数据");
    }
}
```

代理对象：` ProxyFactory `：

```java
import java.lang.reflect.Method;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

// 实现一个MethodInterceptor，方法调用会被转发到该类的intercept()方法。
public class ProxyFactory implements MethodInterceptor{
    private Object target;//维护一个目标对象
    
    public ProxyFactory(Object target) {
        this.target = target;
    }
    
    //为目标对象生成代理对象
    public Object getProxyInstance() {
        //工具类
        Enhancer en = new Enhancer();
        //设置父类（即指定要被代理的目标对象）
        en.setSuperclass(target.getClass());
        //设置回调函数（即指定实际处理逻辑的代理对象）
        en.setCallback(this);
        //创建子类对象代理。对这个对象所有非final方法的调用都会转发给MethodInterceptor.intercept()方法
        return en.create();
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("开启事务");
        // 执行目标对象的方法
        Object returnValue = method.invoke(target, args);
        System.out.println("关闭事务");
        return null;
    }
}
```

测试类：

```java
import org.junit.Test;

public class TestProxy {

    @Test
    public void testCglibProxy(){
        //目标对象
        UserDao target = new UserDao();
        System.out.println(target.getClass());
        //代理对象
        UserDao proxy = (UserDao) new ProxyFactory(target).getProxyInstance();
        System.out.println(proxy.getClass());
        //执行代理对象方法
        proxy.save();
    }
}
```

输出：

```java
class com.cglib.UserDao
class com.cglib.UserDao$$EnhancerByCGLIB$$552188b6
开启事务
保存数据
关闭事务
```

## 总结

JDK原生动态代理是Java原生支持的，不需要任何外部依赖，但是它只能基于接口进行代理；CGLIB通过继承的方式进行代理，无论目标对象有没有实现接口都可以代理，但是无法处理final的情况。

## 参考资料

1. [Java 动态代理详解](https://juejin.im/post/5c1ca8df6fb9a049b347f55c)

2. [Java三种代理模式：静态代理、动态代理和cglib代理](https://segmentfault.com/a/1190000011291179)

3. [Java动态代理InvocationHandler和Proxy学习笔记](https://blog.csdn.net/yaomingyang/article/details/80981004)

4. [Java Proxy和CGLIB动态代理原理](https://www.cnblogs.com/CarpenterLee/p/8241042.html)