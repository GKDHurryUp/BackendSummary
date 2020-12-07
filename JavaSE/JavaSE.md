#  JavaSE #

## 基本数据类型

整形：

byte（1字节）、short（2字节）、int（4字节）、long（8字节）

浮点型：

float（4字节）、double（8字节）

布尔型：

boolean(1位)

字符类型：

char（2字节Unicode ）

## Comparable和 Comparator 的区别 ##
1. Comparable接口位于java.lang包，Comparator接口位于java.util包下
2. Comparable是内部比较器，需要比较的对象来实现接口，重写compareTo方法，对象的耦合度高
3. Comparator是外部比较器，需要定义一个比较类去实现接口，重写里面的compare方法，把需要比较的对象传入参数，无需改变类的结构，降低了耦合度，更加灵活。


## 为什么有些接口定义了equals和hashCode等方法？ ##
1. 从技术角度来看,这不是必需的：每个类都从Object类继承这两个方法.将这两种方法添加到接口技术上不会增加​​任何内容
2. 从实现角度来看：这些定义不会“覆盖”Object类的定义.因此,无需编写自己的实现即可实现此接口
3. 从文档的角度来看：Map.Entry.equals()和Map.Entry.hashCode()的文档对这些方法在界面的任何实现中应该做什么都有非常具体的要求（为了用工具生成文档描述）

## 你怎么理解多态？

面向对象三大特性，封装、继承、多态。因为封装好了才能继承，封装和继承都是为多态做准备的。多态就是同一个接口，使用不同的实例而执行不同操作

三个前提条件：

1. **继承关系**

2. **方法的重写**

3. **向上转型（即父类引用指向子类对象）**

- 编译时多态
  - 编译期间决定目标方法
  - 通过overload重载实现
  - 方法名相同，参数不同（==返回类型无关==，调用相同的方法，编译器不知道使用哪个）
- 运行时多态
  - 运行期间决定目标方法
  - 同名同参
  - 在继承时，override重写实现
  - JVM决定目标方法

### 运行时多态实现机制

方法调用

在class文件中，存放了方法表结构，方法表中存放了**方法入口的地址**，==父类方法表在子类的方法表中的位置是一样的==，JVM调用实例方法只需要指定offset偏移量，调用方法表中的第几个方法即可。在执行时，需要先查看运行时常量池，找到方发表的偏移量，通过invokeVirtual指令对应方法区中不同的偏移量执行不同的方法，实现运行时多态

接口调用 =

Java 类是可以同时实现多个接口，同样的方法在基类和派生类的方法表的位置就可能不一样了，因此对于接口调用需要通过invokeinterface指令，==搜索方法表==，找到合适的方法地址 

## 类型擦除

Java语言的泛型采用的是**擦除法**实现的**伪泛型**，泛型类型只有在**静态类型检查期间**才出现，泛型信息（类型变量、参数化类型）编译之后通通被除掉了，替换成它们非泛型上界

使用擦除法的好处就是实现简单、非常容易Backport，运行期也能够节省一些类型所占的内存空间。

而擦除法的坏处就是，通过这种机制实现的泛型远不如真泛型灵活和强大。

## Cglib和jdk动态代理的区别

1、Jdk动态代理：利用拦截器（必须实现InvocationHandler）加上**反射机制**生成一个代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理

2、Cglib动态代理：利用ASM框架，对代理对象类生成的**class文件**加载进来，**通过修改其字节码生成子类**来处理

## 抽象类和接口的区别

关键字不同，声明：抽象类用 abstract class  ,  接口用 interface

继承：抽象类用 extends  ,  接口用 implements

**从语法层面来说**

1. 抽象类可以提供成员方法的实现细节，而接口中只能存在抽象方法（jdk1.8之后：接口中可以有default和static方法）
2. 抽象类中成员变量可以是多种类型，接口中成员变量必须用public，static，final修饰
3. 抽象类拥有构造方法，而接口没有构造方法
4. 一个类只能继承一个抽象类，但可以实现多个接口
5. 抽象类中允许含有静态代码块和静态方法，接口不能

**从设计层面来说**

1. 抽象类是对共同一个类的属性，行为等方面进行抽象，而接口则是对行为抽象
2. 抽象类可以类比为模板，而接口可以类比为协议

## 如何定一个注解？

创建自定义注解键字前有个@符号，在类上加入四种元注解，

- @Documented 

  表示使用该注解的元素应被javadoc或类似工具文档化

- @Target

  表示支持注解的程序元素的种类TYPE, METHOD, CONSTRUCTOR, FIELD

- @Inherited

  表示一个注解类型会被自动继承

- @Retention

  表示注解类型保留时间的长短，可能的值有SOURCE, CLASS, 以及RUNTIME。


## 枚举类

使用一个enum关键字来定义枚举类型，在进行编译之后，编译器会生成一个相关的类并使用final修饰，继承了Java API中的java.lang.Enum类，我们定义的枚举类，都是使用 public static final 来修饰。

## String为什么是final？

String基本约定中最重要的一条是immutable，如果没有声明为final，那么StringChilld就有可能是被复写为mutable的，这样就打破了成为共识的基本约定。

假如我继承了一个String，重写里面的方法并且改为mutable方式，那么所有依赖于这个方法的行为就有可能错乱

## String，StringBuilder和StringBuffer的区别

1. **String**使用final关键字修饰

   String内部是final修饰的char[] value，字符数组的长度你定义多少，就是多少，不存在字符数组扩容

2. **StringBuilder**（非线程安全）

   StringBuilder 类内部维护可变长度char[] ， 初始化数组容量为16，存在扩容， 其append拼接字符串方法内部调用System的native方法，进行数组的拷贝，不会重新生成新的StringBuilder对象。

   调用toString方法，会执行String的构造函数，会进行一次char[]的copy操作，不会共享StringBuilder对象内部的char[]

3. **StringBuffer**（线程安全的）

   基本上与StringBuilder一致，但其为线程安全的字符串操作类，大部分方法都采用了Synchronized关键字修改，以此来实现在多线程下的操作字符串的安全性

   其toString方法而重新生成的String对象，会共享StringBuffer对象中的toStringCache属性（char[]），但是每次的StringBuffer对象修改，都会置null该属性值。

## new Integer(1) 和Integer.valueOf(1)的区别是什么？

构造函数会直接创建一个Integer对象

```Java
public Integer(int value) {
	this.value = value;
}
```

调用valueOf，如果i在-128~127之间，会返回缓冲池数组中维护的对象

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

### Integer 和 Long 的 hashCode() 方法实现有什么区别？

hashCode是int值，对于Integer直接返回对应的value

而对于Long的话，要讲高32位和低32位做计算，返回(int)(value ^ (value >>> 32))

## equals()和hashCode()原理

equals比较两个对象时，首先先去判断两个对象是否具有相同的地址，如果是同一个对象的引用，则直接返回true；

如果地址不一样，则证明不是引用同一个对象，接下来就是挨个去比较两个字符串对象的内容是否一致，完全相等返回true，否则false

## 自定义对象作为HashMap的key需要注意什么？

Object类的hashCode()方法返回这个对象存储的内存地址的编号，而equals()比较的是内存地址是否相等。

因此用自定义类作为key，必须重写equals()和hashCode()方法。

## 重写equals()和hashCode()方法有什么原则？

equals():

1. 自反性
2. 对称性
3. 传递性
4. 一致性

hashCode():

1.  只要对象equals方法涉及到的关键域内容不改变，那么这个对象的hashCode总是返回相同的整数
2. 如果两个对象的equals(Object obj)方法时相等的，那么调用这两个对象中的任意一个对象的hashCode方法必须产生相同的整数结果。如果两个对象equals方法不同，那么必定返回不同的hashCode整数结果。

## 异常体系

![](..\imgs\异常体系.png)

## I/O流

按类型分：

- 字符流
- 字节流

按流向分：

- 输入流
- 输出流

![](..\imgs\JAVA_IO流.jpg)

## lambda表达式

### 什么是Lambda？

Lambda是JDK1.8添加的新特性，实际上就是一个匿名函数

### 为什么要使用Lambda？

使用Lambda表达式可以对一个接口进行非常简洁的实现

### Lambda对接口的要求？

可以使用Lambda表达式对某些接口进行简单的实现，要求定义的抽象方法只能是一个

> JDK1.8 对接口加入新特性：default（没有影响）

**@FunctionalInterface**

修饰函数式接口，接口中的抽象方法只能有一个

### 系统内置函数式接口

> **Predicate**<T> ， 参数T，返回Boolean
>
> **Consumer**<T>， 参数T，返回值void
>
> **Function**<T, R>， 参数T， 返回值R
>
> **Supplier**<T>，      无参， 返回值T
>
> UnaryOperator<T>， 参数T， 返回值T
>
> BiFunction<T, U, R>，参数T，U，返回值R
>
> BinaryOperator<T>， 参数T，T，返回值T

### 闭包问题

在lambda表达式/匿名类内部，如果引用某局部变量，则直接将其视为final，此时不能对该变量进行修改



## 浅拷贝深拷贝

浅拷贝：对基本数据类型进行值传递，对引用数据类型进行引用传递的拷贝，此为浅拷贝

深拷贝：对基本数据类型进行值传递，对引用数据类型，创建一个新的对象，并复制其内容，此为深拷贝

在 Java 中，所有的 Class 都继承自 Object ，而在 Object 上，存在一个 clone() 方法，调用native系统方法进行拷贝。

直接调用是浅拷贝，如果要完成深拷贝需要

- 重写clone()方法，对其内的引用类型的变量，再进行一次 clone()
- 序列化，再反序列化