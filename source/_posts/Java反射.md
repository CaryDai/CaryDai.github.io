---
title: Java反射
date: 2019-12-01 17:07:15
tags: 反射
categories: Java
description:
---

## 概念

反射，可以理解为在**运行时期**获取对象类型信息的操作。传统的编程方法要求程序员在**编译阶段**决定使用的类型，但是在反射的帮助下，编程人员可以动态获取这些信息，从而编写更加具有可移植性的代码。 

Java 反射机制主要提供了以下功能，这些功能都位于 `java.lang.reflect`

- 在运行时判断任意一个对象所属的类。
- 在运行时构造任意一个类的对象。
- 在运行时判断任意一个类所具有的成员变量和方法。
- 在运行时调用任意一个对象的方法。
- 生成动态代理。

众所周知，所有 Java 类均继承了 Object 类，在 Object 类中定义了一个 getClass() 方法，该方法返回同一个类型为 Class 的对象。例如，下面的示例代码：

```java
Class labelCls=label1.getClass();    //label1为 JLabel 类的对象
```

利用 Class 类的对象 labelCls 可以访问 labelCls 对象的描述信息、JLabel 类的信息以及基类 Object 的信息。 下图列出了反射可以访问的信息：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%8F%8D%E5%B0%84/1.%E5%8F%8D%E5%B0%84%E5%8F%AF%E8%AE%BF%E9%97%AE%E7%9A%84%E5%B8%B8%E7%94%A8%E4%BF%A1%E6%81%AF.png)

注意： 在调用 getFields() 和 getMethods() 方法时将会依次获取权限为 public 的字段和变量，然后将包含从超类中继承到的成员实量和方法。而通过 getDeclareFields() 和 getDeclareMethod() 只是获取在本类中定义的成员变量和方法。 

## 访问构造方法

为了能够**动态获取** 对象构造方法的信息，首先需要通过下列方法之一创建一个 `Constructor` 类型的对象或者数组。 

- getConstructors()
- getConstructor(Class<?>…parameterTypes)
- getDeclaredConstructors()
- getDeclaredConstructor(Class<?>...parameterTypes)

如果是访问指定的构造方法，需要根据该构造方法的入口参数的类型来访问。例如，访问一个入口参数类型依次为 int 和 String 类型的构造方法，下面的两种方式均可以实现。

```java
objectClass.getDeclaredConstructor(int.class,String.class);
objectClass.getDeclaredConstructor(new Class[]{int.class,String.class});
```

 创建的每个 Constructor 对象表示一个构造方法，然后利用 Constructor 对象的方法操作构造方法。Constructor 类的常用方法如下图所示。 

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%8F%8D%E5%B0%84/2.Constructor%E7%B1%BB%E7%9A%84%E5%B8%B8%E7%94%A8%E6%96%B9%E6%B3%95.png)

下面看一个例子：

 (1) 首先创建一个 Book 类表示图书信息。在该类中声明一个 String 型变量表示图书名称，两个 int 型变量分别表示图书编号和价格，并提供 3 个构造方法。 

```java
public class Book {
    String name; // 图书名称
    int id, price; // 图书编号和价格

    // 空的构造方法
    private Book() {
    }

    // 带两个参数的构造方法
    protected Book(String _name, int _id) {
        this.name = _name;
        this.id = _id;
    }

    // 带可变参数的构造方法
    public Book(String... strings) throws NumberFormatException {
        if (0 < strings.length)
            id = Integer.valueOf(strings[0]);
        if (1 < strings.length)
            price = Integer.valueOf(strings[1]);
    }

    // 输出图书信息
    public void print() {
        System.out.println("name=" + name);
        System.out.println("id=" + id);
        System.out.println("price=" + price);
    }
}
```

 (2) 编写测试类 Test01，在该类的 main() 方法中通过反射访问 Book 类中的所有构造方法，并将该构造方法是否带可变类型参数、入口参数类型和可能拋出的异常类型信息输出到控制台。 

```java
import java.lang.reflect.Constructor;

public class Test01 {
    public static void main(String[] args) {
        // 获取动态类Book
        Class<?> book = Book.class;
        // 获取Book类的所有构造方法
        Constructor<?>[] declaredContructors = book.getDeclaredConstructors();
        // 遍历所有构造方法
        for (int i = 0; i < declaredContructors.length; i++) {
            Constructor<?> con = declaredContructors[i];
            // 判断构造方法的参数是否可变
            System.out.println("查看是否允许带可变数量的参数：" + con.isVarArgs());
            System.out.println("该构造方法的入口参数类型依次为：");
            // 获取所有参数类型
            Class<?>[] parameterTypes = con.getParameterTypes();
            for (int j = 0; j < parameterTypes.length; j++) {
                System.out.println(" " + parameterTypes[j]);
            }
            System.out.println("该构造方法可能拋出的异常类型为：");
            // 获取所有可能拋出的异常类型
            Class<?>[] exceptionTypes = con.getExceptionTypes();
            for (int j = 0; j < exceptionTypes.length; j++) {
                System.out.println(" " + parameterTypes[j]);
            }
            // 创建一个未实例化的Book类实例
            Book book1 = null;
            while (book1 == null) {
                try { // 如果该成员变量的访问权限为private，则拋出异常
                    if (i == 1) {
                        // 通过执行带两个参数的构造方法实例化book1
                        book1 = (Book) con.newInstance("Java 教程", 10);
                    } else if (i == 2) {
                        // 通过执行默认构造方法实例化book1
                        book1 = (Book) con.newInstance();
                    } else {
                        // 通过执行可变数量参数的构造方法实例化book1
                        Object[] parameters = new Object[] { new String[] { "100", "200" } };
                        book1 = (Book) con.newInstance(parameters);
                    }
                } catch (Exception e) {
                    System.out.println("在创建对象时拋出异常，下面执行 setAccessible() 方法");
                    con.setAccessible(true); // 设置允许访问 private 成员
                }
            }
            book1.print();
            System.out.println("=============================\n");
        }
    }
}
```

输出：

```
查看是否允许带可变数量的参数：true
该构造方法的入口参数类型依次为：
 class [Ljava.lang.String;
该构造方法可能拋出的异常类型为：
 class [Ljava.lang.String;
name=null
id=100
price=200
=============================

查看是否允许带可变数量的参数：false
该构造方法的入口参数类型依次为：
 class java.lang.String
 int
该构造方法可能拋出的异常类型为：
name=Java 教程
id=10
price=0
=============================

查看是否允许带可变数量的参数：false
该构造方法的入口参数类型依次为：
该构造方法可能拋出的异常类型为：
在创建对象时拋出异常，下面执行 setAccessible() 方法
name=null
id=0
price=0
=============================
```

## 访问方法

要**动态获取** 一个对象方法的信息，首先需要通过下列方法之一创建一个 `Method` 类型的对象或者数组。 

- getMethods()
- getMethods(String name,Class<?> …parameterTypes)
- getDeclaredMethods()
- getDeclaredMethods(String name,Class<?>...parameterTypes)

如果是访问指定的构造方法，需要根据该方法的入口参数的类型来访问。例如，访问一个名称为 max,入口参数类型依次为 int 和 String 类型的方法。下面的两种方式均可以实现：

```java
objectCiass.getDeclaredConstructor("max",int.class,String.class);
objectClass.getDeclaredConstructor("max",new Ciass[]{int.class,String.class});
```

Method 类的常用方法如下图所示：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%8F%8D%E5%B0%84/3.Method%E7%B1%BB%E7%9A%84%E5%B8%B8%E7%94%A8%E6%96%B9%E6%B3%95.png)

下面看一个例子：

 (1) 首先创建一个 Book1 类，并编写 4 个具有不同作用域的方法。 

```java
public class Book1 {
    // static 作用域方法
    static void staticMethod() {
        System.out.println("执行staticMethod()方法");
    }

    // public 作用域方法
    public int publicMethod(int i) {
        System.out.println("执行publicMethod()方法");
        return 100 + i;
    }

    // protected 作用域方法
    protected int protectedMethod(String s, int i) throws NumberFormatException {
        System.out.println("执行protectedMethod()方法");
        return Integer.valueOf(s) + i;
    }

    // private 作用域方法
    private String privateMethod(String... strings) {
        System.out.println("执行privateMethod()方法");

        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < sb.length(); i++) {
            sb.append(strings[i]);
        }
        return sb.toString();
    }
}
```

 (2) 编写测试类 Test02，在该类的 main() 方法中通过反射访问 Book1 类中的所有方法，并将该方法是否带可变类型参数、入口参数类型和可能拋出的异常类型信息输出到控制台。 

```java
import java.lang.reflect.Method;

public class Test02 {
    public static void main(String[] args) {
        // 获取动态类Book1
        Book1 book = new Book1();
        Class<?> class1 = book.getClass();
        // 获取Book1类的所有方法
        Method[] declaredMethods = class1.getDeclaredMethods();
        for (int i = 0; i < declaredMethods.length; i++) {
            Method method = declaredMethods[i];
            System.out.println("方法名称为：" + method.getName());
            System.out.println("方法是否带有可变数量的参数：" + method.isVarArgs());
            System.out.println("方法的参数类型依次为：");
            // 获取所有参数类型
            Class<?>[] methodType = method.getParameterTypes();
            for (int j = 0; j < methodType.length; j++) {
                System.out.println(" " + methodType[j]);
            }
            // 获取返回值类型
            System.out.println("方法的返回值类型为：" + method.getReturnType());
            System.out.println("方法可能抛出的异常类型有：");
            // 获取所有可能抛出的异常
            Class<?>[] methodExceptions = method.getExceptionTypes();
            for (int j = 0; j < methodExceptions.length; j++) {
                System.out.println(" " + methodExceptions[j]);
            }
            boolean isTurn = true;
            while (isTurn) {
                try { // 如果该成员变量的访问权限为private，则抛出异常
                    isTurn = false;
                    if (method.getName().equals("staticMethod")) { // 调用没有参数的方法
                        method.invoke(book);
                    } else if (method.getName().equals("publicMethod")) { // 调用一个参数的方法
                        System.out.println("publicMethod(10)的返回值为：" + method.invoke(book, 10));
                    } else if (method.getName().equals("protectedMethod")) { // 调用两个参数的方法
                        System.out.println("protectedMethod(\"10\",15)的返回值为：" + method.invoke(book, "10", 15));
                    } else if (method.getName().equals("privateMethod")) { // 调用可变数量参数的方法
                        Object[] parameters = new Object[] { new String[] { "J", "A", "V", "A" } };
                        System.out.println("privateMethod()的返回值为：" + method.invoke(book, parameters));
                    }
                } catch (Exception e) {
                    System.out.println("在设置成员变量值时抛出异常，下面执行setAccessible()方法");
                    method.setAccessible(true); // 设置为允许访问private方法
                    isTurn = true;
                }
            }
            System.out.println("=============================\n");
        }
    }
}
```

输出：

```
方法名称为：staticMethod
方法是否带有可变数量的参数：false
方法的参数类型依次为：
方法的返回值类型为：void
方法可能抛出的异常类型有：
执行staticMethod()方法
=============================

方法名称为：publicMethod
方法是否带有可变数量的参数：false
方法的参数类型依次为：
 int
方法的返回值类型为：int
方法可能抛出的异常类型有：
执行publicMethod()方法
publicMethod(10)的返回值为：110
=============================

方法名称为：privateMethod
方法是否带有可变数量的参数：true
方法的参数类型依次为：
 class [Ljava.lang.String;
方法的返回值类型为：class java.lang.String
方法可能抛出的异常类型有：
在设置成员变量值时抛出异常，下面执行setAccessible()方法
执行privateMethod()方法
privateMethod()的返回值为：
=============================

方法名称为：protectedMethod
方法是否带有可变数量的参数：false
方法的参数类型依次为：
 class java.lang.String
 int
方法的返回值类型为：int
方法可能抛出的异常类型有：
 class java.lang.NumberFormatException
执行protectedMethod()方法
protectedMethod("10",15)的返回值为：25
=============================
```

## 访问成员变量

通过下列任意一个方法访问成员变量时将返回 Field 类型的对象或数组。

- getFields()
- getField(String name)
- getDeclaredFields()
- getDeclaredField(String name)

Field 类的常用方法如下图所示：

![](https://markdown-1259486229.cos.ap-shanghai.myqcloud.com/%E5%8F%8D%E5%B0%84/4.Field%E7%B1%BB%E7%9A%84%E5%B8%B8%E7%94%A8%E6%96%B9%E6%B3%95.png)

下面看一个例子：

 (1) 首先创建一个 Book2 类，在该类中依次声明一个 String、int、float 和 boolean 类型的成员，并设置不同的访问作用域。 

```java
public class Book2 {
    String name;
    public int id;
    private float price;
    protected boolean isLoan;
}
```

 (2) 编写测试类 Test03，在该类的 main() 方法中通过反射访问 Book2 类中的所有成员，并将该成员的名称和类型信息输出到控制台。 

```java
import java.lang.reflect.Field;

public class Test03 {
    public static void main(String[] args) {
        Book2 book = new Book2();
        // 获取动态类Book2
        Class<?> class1 = book.getClass();
        // 获取Book2类的所有成员
        Field[] declaredFields = class1.getDeclaredFields();
        // 遍历所有的成员
        for (int i = 0; i < declaredFields.length; i++) { // 获取类中的成员变量
            Field field = declaredFields[i];
            System.out.println("成员名称为：" + field.getName());
            Class<?> fieldType = field.getType();
            System.out.println("成员类型为：" + fieldType);
            boolean isTurn = true;
            while (isTurn) {
                try { // 如果该成员变量的访问权限为private，则抛出异常
                    isTurn = false;
                    System.out.println("修改前成员的值为：" + field.get(book));
                    // 判断成员类型是否为int
                    if (fieldType.equals(int.class)) {
                        System.out.println("利用setInt()方法修改成员的值");
                        field.setInt(book, 100);
                    } else if (fieldType.equals(float.class)) { // 判断成员变量类型是否为float
                        System.out.println("利用setFloat()方法修改成员的值");
                        field.setFloat(book, 29.815f);
                    } else if (fieldType.equals(boolean.class)) { // 判断成员变量是否为boolean
                        System.out.println("利用setBoolean()方法修改成员的值");
                        field.setBoolean(book, true);
                    } else {
                        System.out.println("利用set()方法修改成员的值");
                        field.set(book, "Java编程");
                    }
                    System.out.println("修改后成员的值为：" + field.get(book));
                } catch (Exception e) {
                    System.out.println("在设置成员变量值时抛出异常，下面执行setAccessible()方法");
                    field.setAccessible(true);
                    isTurn = true;
                }
            }
            System.out.println("=============================\n");
        }
    }
}
```

输出：

```
成员名称为：name
成员类型为：class java.lang.String
修改前成员的值为：null
利用set()方法修改成员的值
修改后成员的值为：Java编程
=============================

成员名称为：id
成员类型为：int
修改前成员的值为：0
利用setInt()方法修改成员的值
修改后成员的值为：100
=============================

成员名称为：price
成员类型为：float
在设置成员变量值时抛出异常，下面执行setAccessible()方法
修改前成员的值为：0.0
利用setFloat()方法修改成员的值
修改后成员的值为：29.815
=============================

成员名称为：isLoan
成员类型为：boolean
修改前成员的值为：false
利用setBoolean()方法修改成员的值
修改后成员的值为：true
=============================
```

## .class, class.forName(), getClass()区别

### Class 类概念

Class 也是一个 Java 类，保存的是与之对应 Java 类的 meta信息（元信息），用来描述这个类的结构，比如描述一个类有哪些成员，有哪些方法等，一般在反射中使用。

Java 源程序（.java 文件）在经过 Java 编译器编译之后就被转换成 Java 字节码（.class 文件）。类加载器负责读取 Java 字节码，并转换成 java.lang.Class类的一个实例（Class 对象）。也就是说，在 Java 中，每个 java 类都有一个相应的 Class 对象，用于表示这个 java 类的类型信息。

### 类加载概念

当我们使用一个类的时候（比如 new 一个类的实例），JVM会检查此类是否被加载到内存，如果没有，则会执行加载操作。Java 也提供了手动加载类的接口，class.forName()方法就是其中之一。

### 类加载器的概念

类加载器负责读取 Java 字节代码，并转换成 java.lang.Class 类的一个实例。每个这样的实例用来表示一个 Java 类。通过此实例的 newInstance() 方法就可以创建出该类的一个对象。基本上所有的类加载器都是 java.lang.ClassLoader 类的一个实例。

```java
// Class.newInstance()
HelloWorld helloWorld1 = (HelloWorld) Class.forName("TestNewInstance.HelloWorld").newInstance();
HelloWorld helloWorld2 = HelloWorld.class.newInstance();

// Constructor.newInstance()
HelloWorld helloWorld3 = HelloWorld.class.getConstructor(String.class).newInstance("ZTH");
```

### 类初始化概念

类被加载之后，jvm 已经获得了一个描述类结构的 Class 实例。但是还需要进行类初始化操作之后才能正常使用此类，类初始化操作就是执行一遍类的静态语句，包括静态变量的声明还有静态代码块。

现在回到那三种方法创建类的区别:

1. Class.forName()在运行时加载；Class.class和getClass()是在编译时加载.

## 参考资料

1. [java中Class对象详解和类名.class, class.forName(), getClass()区别](https://www.cnblogs.com/Seachal/p/5371733.html) 
2. [通过Class.newInstance()和Constructor.newInstance()两种反射方法创建对象的异同](https://segmentfault.com/a/1190000009174512)
3. [Java class.forname 详解](https://www.runoob.com/w3cnote/java-class-forname.html)