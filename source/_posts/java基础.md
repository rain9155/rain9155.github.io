---
title: java学习总结之基础
tags: java
categories: java
date: 2019-07-19 12:24:58
---


## 前言
万事开头难，准备从零把java相关知识点捡起来，把自己所学的Java知识点归纳，下面是关于java的一些基本知识点。

<!--more-->

## java代码的运行过程
{% asset_img java1.png java1 %}

* 创建java源程序，扩展名为.java
* 使用javac命令编译源程序为字节码文件，扩展名为.class
* 使用java命令运行字节码文件，在不同平台执行

## 数据类型

下面用一张表概括：

|   数据类型   |    类型说明符    | 位数 | 字节 |
| :---: | :---: | :---: | :---: |
|     整形     |       int        |  32  |  4   |
|    短整型    |      short       |  16  |  2   |
|    长整形    |       long       |  64  |  8   |
|    字节型    |       byte       |  8   |  1   |
| 单精度浮点型 |      float       |  32  |  4   |
| 双精度浮点型 |      double      |  64  |  8   |
|   字符类型   |       char       |  16  |  2   |
|   布尔类型   |     boolean      |  -   |  -   |
|  字符串类型  |      String      |  -   |  -   |
|  自定义类型  | public class ... |  -   |  -   |
其中java的数据类型又分为：
* 基本类型/原始类型（primitive type）：用来保存简单的单个数据，如：int、short、long、byte、float、double、boolean、char共8种。

* 类类型/引用类型（class type or reference types）：用来保存复杂的组合数据，如String和自定义类型。

在java中，char类型实际是一个16位的无符号整数（<=65535），可以保存中文和转义字符(`\b`,`\t`,`\n`等)。

而在java中并没有明确的表示boolean类型应该占多少位，其大小与JVM实现相关，JVM的建议如下：

1、boolean 类型应该被编译成 int 类型来使用，占 4 个字节；

2、boolean 数组应该被编译成 byte 数组类型，每个 boolean 数组成员占 1 个 字节.

可以肯定的是，boolean 类型不会只占 1 个位，boolean 类型的大小与JVM实现相关。

## 成员变量和局部变量
```java
public class 类名{

    数据类型 变量1；//成员变量
    
    返回值类型 方法名(){
    
        数据类型 变量2；//局部变量
        
    }
    
}
```
1、成员变量的作用域在整个类都是可见的。
2、局部变量的作用域仅在定义它们的方法中可见。
3、成员变量有默认的初始值（数字为0，对象为null）。
4、局部变量没有默认的初始值，需要赋初值再使用。
5、成员变量和局部变量重名时，局部变量的优先级更高。

## 字符串
### 1、String类型
在java中String是引用类型，它的构造器如下：

{% asset_img java2.png java2 %}

除了用以上new的方法创建一个字符串，还可以用以下方式：

```java
String name1 = "rain";
String name2 = "rain";
```
那么用`new`和用`=`有什么不同的呢? new出来字符串是一个String对象，它被放在堆中，地址不一样。用=赋值的字符串是从字符串池（String Pool，保存着所有字符串字面量，这些字面量在编译时期就确定）中拿的，如果这个字符串在池中没有，就会先放进池，所以上面两个name1和name2是同一个字符串。

**String中常用的方法：**

{% asset_img java3.png java3 %}

**String的比较：**

{% asset_img java4.png java4 %}

**获取String的字串的方法：**

{% asset_img java5.png java5 %}

在 Java 8 中，String 内部使用 char 数组存储数据，在 Java 9 之后，String 类的实现改用 byte 数组存储字符串，这些数组在都用final修饰，所以才保证String是**不可变**的，不可变就是每次意图修改String，都会产生一个新的String对象。

> 1、在字符串的比较中`==`是用来比较地址的，String的equals方法才是用来比较两个字符串是否相等 ；
> 2、在 Java 7 之前，String Pool 被放在运行时常量池中，它属于永久代，而在 Java 7，String Pool 被移到堆中。这是因为永久代的空间有限，在大量使用字符串的场景下会导致 OutOfMemoryError 错误。

### 2、StringBuffer和StringBuilder
与String类不同的是，StringBuffer和StringBuilder是**可变**的，它们的对象可以被多次修改，因为它里面的数组并没有使用final修饰，所以每次意图修改StringBuffer或StringBuilder对象时，都会在原始对象上进行修改，不会产生新的对象，StringBuilder是在JDK5中被提出来的，它和StringBuffer之间最大的不同是StringBuilder的方法都不是线程安全的，而StringBuffer的方法都是线程安全的。

**StringBuffer的构造方法（StringBuilder类似）：**

{% asset_img java6.png java6 %}

**StringBuffer的常用方法：**

{% asset_img java7.png java7 %}

### 3、字符串的拼接

在java中，可以通过**+**、String的**concat方法**、StringBuilder的**append**方法和StringBuffer的**append**方法来拼接字符串，如下：

**使用+拼接：**

```java
public static void main(String[] args) {
    String str1 = "Hello";
    String str2 = "World!";
    String str = str1 + " " + str2;
}
```

反编译后，如下：

```java
public static void main(String[] args) {
    String str1 = "Hello";
    String str2 = "World!";
    String str = (new StringBuilder()).append(str1).append(" ").append(str2).toString();
}
```

可以看到使用**+**拼接字符串时，底层会new一个StringBuilder对象，调用StringBuilder的append方法来拼接。

**使用String的concat方法拼接：**

```java
public static void main(String[] args) {
    String str1 = "Hello";
    String str2 = "World!";
    String str = str1.concat(" ").concat(str2);
}
```

查看String的concat方法的源码，如下：

```java
public String concat(String str) {
    int olen = str.length();
    if (olen == 0) {
        return this;
    }
  	//...
    int len = length();
    //创建一个新的长度的字节数组，新长度 = 原始字符串长度len + 要拼接字符串长度olen
    byte[] buf = StringUTF16.newBytesFor(len + olen);
    //把原始字符串的字节数组复制到buf中
    getBytes(buf, 0, UTF16);
    //把要拼接的字符串的字节数组复制到buf中
    str.getBytes(buf, len, UTF16);
    //通过buf创建一个String返回
    return new String(buf, UTF16);
}
```

可以看到concat方法底层是新创建一个字节数组，长度为原始字符串长度和要拼接字符串长度之和，然后把原始字符串和要拼接字符串的字节数组复制到新的字节数组中，最后通过新的字节数组创建一个新的String对象返回，所以String的concat方法最终会返回一个新的String对象，这也说明了String的**不可变性**，不会修改原始字符串的字节数组到达拼接字符串的目的。

**使用StringBuilder和StringBuffer的append方法拼接：**

```java
//以StringBuilder举例，StringBuffer类似
public static void main(String[] args) {
    String str1 = "Hello";
    String str2 = "World!";
    StringBuilder str = new StringBuilder();
    str.append(str1).append(" ").append(str2).toString();
}
```

查看StringBuilder的append方法的源码，如下：

```java
public StringBuilder append(String str) {
    super.append(str);
    return this;
}

//StringBuilder的父类AbstractStringBuilder中的append方法
public AbstractStringBuilder append(String str) {
    if (str == null) {
        return appendNull();
    }
    int len = str.length();
    //扩容内部的字节数组，扩容后长度 = 原始字符串长度count + 要拼接的字符串长度len
    ensureCapacityInternal(count + len);
    //把要拼接的字符串追加到内部的字节数组
    putStringAt(count, str);
    count += len;
    return this;
}
```

可以看到，StringBuilder的append方法会直接拷贝待拼接的字符串字节数组到**内部的字节数组**中，如果内部的字节数组长度不够，就会先扩容后再拷贝，所以append方法并不会产生新的StringBuilder对象，对于StringBuffer的append方法，它和StringBuilder的append方法的逻辑一样，只是多了一个synchronized关键字。

**效率比较：**

分别使用+、concat方法、StringBuffer和StringBuilder的append方法来拼接字符串，各自的效率怎么样？在单线程环境下的一个for循环中拼接大量字符串，经过测试，它们的效率高低如下：

**StringBuilder > StringBuffer > concat > +**

+效率最低，这是因为每次拼接字符串时，都会new一个StringBuilder对象来拼接，频繁的新建对象是很耗时的，而StringBuffer每次append都需要进行同步，所以它的效率比StringBuilder低。

> 阿里巴巴Java开发手册建议：在循环体内，使用 `StringBuilder` 的 `append` 方法进行字符串的拼接，而不要使用`+`，因为`+`会频繁的创建新的`StringBuilder`对象导致耗费更多的时间和造成内存资源的浪费。

## 包装类
包装类就是将java的基本数据类型打包成对象处理，包装类都在java.lang包中，下面用一个表显示：

| 基本数据类型 |  包装类   |
| :---: | :---: |
|     int      | Interger  |
|    short     |   Short   |
|     long     |   Long    |
|     char     | Character |
|     byte     |   Byte    |
|    float     |   Float   |
|    double    |  Double   |
|   boolean    |  Boolean  |
它涉及到以下两种操作：
### 1、装箱(boxing)
以double装箱为例：
```java
double num = 3.14；
Double dnum1 = new Double(num);//1
Double dnum2 = Double.valueOf(num);//2
Double dnum3 = num;//3
```
注释1、2都是手动装箱，注释3是自动装箱。
### 2、拆箱(unboxing)
同样上述double拆箱为例：
```java
num = dnum1;//1
num = dnum2.doubleValue();//2
```
注释1是自动拆箱，注释2是手动拆箱。

除了Double和Float，每个包装类都会有一个默认大小的缓存池，例如Integer，缓存池默认大小是-128-127，缓存池中都是缓存了一些经常使用的值，而对于Double和Float，都是浮点型，它们没有经常使用的值，编译器会在自动装箱过程中会调用 valueOf() 方法，因此多个值相同，且值在缓存池范围内的包装类实例使用自动装箱来创建，那么就会引用相同的对象。

> ps:
> 1、包装类没有无参构造，所有包装类的实例都是不可变的，一旦创建对象后，它们的内部值就不能再改变。
> 2、每个基本类型包装类都有常量MAX_VALUE和MIN_VALUE。

## 关键字

### 1、final

防止扩展和重写。

* 修饰成员变量：常量（可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量），不可更改（对于基本数据类型，final使数值不能改变，对于引用类型，final使引用不能改变，即不能引用其他对象，但引用本身可以更改）
* 修饰方法：不可被重写
* 修饰类：不可被继承

### 2、static

可以通过类名直接访问它修饰的属性，静态属性和方法都是优先于类的实例存在。

* 修饰变量：称为静态变量（区别于实例变量）、类变量，类的所有实例都共享静态变量，静态变量在内存中只存在一份
* 修饰方法：称为静态方法，静态方法必须有实现，它不依赖于任何实例，静态方法中只能调用类的静态属性和静态方法，方法中不能有 this 和 super 关键字
* 修饰语句块：称为静态语句块，在类初始化时运行一次
* 修饰内部类：称为静态内部类，非静态内部类依赖于外部类的实例，而静态内部类不需要

存在继承的情况下，初始化顺序为：

父类(静态变量、静态语句块) -> 子类（静态变量、静态语句块） -> 父类（实例变量、普通语句块）-> 父类（构造函数）-> 子类（实例变量、普通语句块） -> 子类（构造函数）

## 关于==、equal()和hashCode()

### 1、==

==是一个关系操作符，所以：

* 如果左右两边的操作数是基本数据类型，那么X==Y，就是判断左右两边的操作数的值是否相等.
* 如果左右两边的操作数是引用数据类型，那么X==Y，就是判断左右两边的操作数的内存地址是否相等.

### 2、equal()

equal()是用来判断两个对象是否等价，即两个对象是否相等，所以如果要重写一个equal方法，需要做到以下3步：

* 先使用**==**判断两个对象的引用是否相等.（地址相同）
* 然后使用**instanceof **判断两个对象是否是同一个类型.（类型相同）
* 最后比较两个对象的内容是否一致.(内容相同)

按照上面3步重写的equal方法，满足自反性、对称性、传递性，如下：

- 自反性：对于非null的x，x.equal(x)返回true；
- 对称性：对于非null的x，y，x.equal(y)返回true当且仅当y.equal(x)返回true；
- 传递性：对于非null的x，y，z，如果x.equal(y)返回true，并且y.equal(z)返回true，那么x.equal(z)返回true。

### 3、hashCode()

hashCode()用来返回一个对象的hash值，它是一个native方法，它主要使用于哈希表中的hash算法，用于定位一个元素的位置，所以当你的对象要作为哈希表中的元素时，你要保证以下几个原则：

* 要比较两个对象是否相等，必须使用equal方法，如果相等，那么调用两个对象的 hashCode 方法必须返回相同的结果，即相等的两个对象返回的hashCode必须是相等的.
* 如果两个对象根据 equals方法比较是不相等的，则 hashCode 方法不一定得返回不同的整数.
* 对同一个对象调用多次hashcode方法必须返回相同的hash值.
* 两个不同对象的hashcode可能相等.
* 两个不同hashcode的对象一定不相等.

在使用hashXX集合添加对象时，集合先调用该对象的hashCode方法，根据哈希函数得到对象在哈希表中的位置，如果该位置没有元素，就直接把它存储在该位置上；如果该位置已经有元素了，就调用对象的equal方法与该位置的每个元素逐个比较，如果相等，就更新该元素，如果都不相等，就把该对象的映射添加到这个位置对应的链表中。

因此在覆盖 equals() 方法时应当总是覆盖 hashCode() 方法，保证等价的两个对象hash值也相等，不然会导致集合中出现重复的元素，一个好的习惯是equals方法中用到的成员变量也必定会在hashcode方法中用到，这样就能保证**两个相等的对象hash值也相等**。

> 当你没有重写hashCode方法时，它可能返回以下的值：
> 1、随机生成数字；
> 2、对象的内存地址，强制转换为int；
> 3、根据对象的内存地址生成；
> 4、1硬编码（用于敏感性测试）；
> 5、一个序列；
> 6、使用线程状态结合xorshift算法生成。
> 具体返回什么需要看不同JDK版本的实现。

## 异常

异常就是一种对象(Exception), 表示阻止程序正常执行的错误。异常类的层次结构如下：

{% asset_img error1.png error1 %}

* 1、RuntimeException和Error以及它们的子类都称为免检异常, 这种异常一般是由程序逻辑错误引起的，对于这种异常，可以选择捕获处理，也可以不处理；
* 2、除了免检异常，其他异常都称为必检异常，这种异常跟程序运行的上下文环境有关，即使程序设计无误，仍然可能因使用的问题而引发，所以Java强制要求我们必须对这些异常进行处理.

由于免检异常可能在程序的任何一个地方出现，为了避免过多的使用try-catch块，java语言不强制要求编写代码捕获免检异常，也不要求在方法头显示声明免检异常。

### 1、常见的异常类型

{% asset_img error2.png error2 %}

### 2、java中的异常处理机制

异常处理机制就是可以使程序处理非预期的情景，并继续正常执行，异常处理机制的主要组成如下：

* try：监控有可能产生异常的语句块
* catch：以合理的方式捕获异常
* finally：不管有没有异常，一定会执行的语句块（一般用来释放资源），除了遇到System.exit(0)语句
* throw：手动引发异常
* throws: 指定由方法引发的异常

所以一个异常捕获处理语句可以如下形式：

```java
try{
    //监控可能产生异常的语句块
}catch(Exception1 e){
    //捕获异常，处理异常，如打印异常信息，日志纪录
}catch(Exception2 e){
    //JDK7后简化写法catch(Exception1|Exception2|Exception3|... e)
}finally{
    //不管有无异常，一定会执行的语句，用来释放资源
}
```

try块中的代码可能会引发多种类型的异常，当引发异常时，会按照catch的顺序进行匹配异常类型，并执行第一个匹配的catch语句。

## 泛型

### 1、为什么使用泛型

在java5之前，任何类型的元素都可以“丢进“集合中，元素一旦进入集合中，元素的类型就会被集合忘记，导致从集合中取出的元素都是**Object**类型，需要进行**强制类型转换**后才能变成我们”丢进“集合前的元素类型，这样就导致了以下两个缺点：

- 1、编程的复杂度增加：任何从集合中取出的元素都要进行强制类型转换，增加编程的工作量；
- 2、运行时容易引发ClassCastException：由于任何类型的元素都可以放进集合中，导致集合中的元素的类型不一致，在取出元素强制类型转换时，就有可能人为地转换错误，引发ClassCastException异常，导致程序崩溃.

所以为了解决集合编译时不检查类型的问题，就出现了**泛型**，泛型(GenericType)是从java5开始支持的新语法，它又被称为参数化类型**ParameterizedType**，ParameterizedType是java5新增的Type，泛型它表示广泛通用的类型，可以在代码编译期就进行**类型检查**，在创建集合的时候可以动态地指明元素的类型是什么，从而约束所有放进集合的元素**类型一致**，这样从集合中取出元素时就**不需要进行强制类型转换**，从而避免了在运行时出现**ClassCastException**异常，如果把错误类型元素放入集合，编译器就会提出**错误**，所以在java中使用泛型时它**保证只要程序在编译期没有提示“UnChecked”未经检查警告，那么运行时就不会产生ClassCastException**。

所以从java5之后，集合框架中的全部类和接口都增加了泛型支持，从而在创建集合时可以动态地指明元素的类型是什么，如：**List\<String\> list = new ArrayList\<String\>()**，其中List\<String\>、ArrayList\<String\>就统称为泛型，而<>括号中的类型就称为**类型形参**。

> java7的时候出现了菱形语法简化了泛型的写法，java7开始允许使用泛型时构造器后面的<>括号内不带类型形参，只需要给出<>括号就行，如：List\<String\> list = new ArrayList<>()。

泛型可以分为**泛型类**(包括类和接口)和**泛型方法**。

### 2、泛型类

泛型类就是直接在类或接口上定义的泛型，上面所讲的集合也是属于泛型类，如下创建一个泛型类：

```java
public class Bean<T> {
    private T t;
    
    public void set(T t) { 
        this.t = t; 
    }
    
    public T get() {
        return t; 
    }
}
```

如果有多个类型形参，在<>括号中就用 **,** 隔开，在创建Bean实例的时候就可以动态的指明**T**(类型形参)的类型，如下：

```java
Bean<String> bean = new Bean<String>();
```

泛型类中的**T**(类型形参)不存在继承的关系，如下是错误的：

```java
Bean<Object> bean = new Bean<String>();//错误，Bean<String>并不是Bean<Object>的子类。
```

同时需要注意静态变量不能使用类型形参修饰，还有静态方法不能使用泛型类声明的类型形参，如下是错误的：

```java
public class Bean<T> {
    private T t;
    private static T t2; //错误，类型形参在编译后会被擦除
    public static void doWork(Bean<T> bean){}//错误，类型形参在编译后会被擦除，如果静态方法需要使用泛型，只能使用泛型方法
}
```

还有instanceof运算符后面不能使用泛型类，如下是错误的：

```java
if(XX instanceof Bean<String>){//错误，不存在Bean<String>对应的Class<String>对象
    //...
}
```

以上错误的原因都可以归结为泛型擦除，不管传入的类型形参是什么类型，在运行时它们总是具有相同的类(Class)。

> 如果从泛型类派生子类时，必须为作为父类的泛型类的类型形参指明类型或者不写<>括号，不写<>括号时泛型类的类型形参默认为上限类型，如果没有上限，默认为Object类型。

### 3、泛型方法

泛型方法就是直接在方法上定义的泛型，如下创建一个泛型方法：

```java
public static <T> T doWork(Bean<T> bean){
    return bean.get();
}
```

如果有多个类型形参，在<>括号中就用 **,** 隔开，在调用doWork方法时，java会自动推断方法形参的类型，从而推断出类型形参的类型，如下：

```java
Bean<String> bean = new Bean<String>();
doWork(bean);
```

上面doWork方法传入形参为bean实例，它的类型形参为String类型，从而推断出doWork方法的类型形参T为String类型。

> 泛型方法允许类型形参被用来表示方法的一个或多个参数之间的类型依赖关系，或者返回值与参数之间的类型依赖关系，如果没有这样的类型依赖关系，就不应该使用泛型方法，可以考虑使用类型通配符。

### 4、类型通配符、上限和下限

类型通配符使用**?**表示，它表示未知的元素类型，当不知道使用什么类型接收时，此时可以使用**?**，如下：

```java
//此时可以往doWork传入List<String>、List<Integer>等实例
public static void doWork(List<?> list){
    for(Object o : list){
        System.out.println(o);
    }
}
```

不管往doWork方法中传入任何类型的List\<XX\>实例，当用**?**接收后，此时List集合中的所有元素类型都是Object，不能往元素类型是**?**的集合中增加、修改元素，只能查询、删除。

类型通配符**?**一般代表所有类型的父类，即Object，可以为**?**添加上限或下限，如下：

```java
//上限：此时的类型形参必须为Number或Number的子类
public static void doWork(List<? extends Number> list){
    for(Number o : list){
        System.out.println(o);
    }
}
```

```java
//下限：此时的类型形参必须为Number或Number的父类
public static void doWork(List<? super Number> list){
    for(Object o : list){
        System.out.println(o);
    }
}
```

在<>中加入extends，就是 **<=** 的关系，在<>中加入super，就是 **>=** 的关系。

> extends和super也可以用于限制泛型的类型形参的上限和下限。

### 5、泛型擦除

泛型其实是一个语法糖，系统并不会为每个泛型生成一个Class对象，它们在运行时始终具有相同的Class对象，例如List\<String\>、List\<Integer\>等泛型类在编译之后，<>括号内的类型信息都会被擦除，在运行时都是使用List的Class对象，并且在反编译后，还是使用强制类型来获取元素，如下：

```java
public static void main(String[] args){
    //类型形参为Integer的泛型
    List<Integer> list = new ArrayList<>();
    list.add(1);
    Integer num = list.get(0);

     //类型形参为String的泛型
    List<String> list2 = new ArrayList<>();
    list2.add("1");
    String str = list2.get(0);
}

//反编译后
public static void main(String args[]) {
    //Integer类型被擦除
    List list = new ArrayList();
    list.add(Integer.valueOf(1));
    Integer num = (Integer)list.get(0);

	//String类型被擦除
    List list2 = new ArrayList();
    list2.add("1");
    String str = (String)list2.get(0);
}
```

还有当把一个具有泛型信息的变量赋值给另一个没有泛型信息的变量时，所有<>括号之间的类型信息都会被擦除，如下：

```java
//类型形参为Integer的泛型
List<Integer> list = new ArrayList<>();
list.add(1);

//把具有Integer类型信息的泛型list赋值给没有泛型信息的list2
List list2 = list;
//此时list的所有类型信息被擦除，变成了Object类型，这是可以往里面添加任何类型的元素
//添加元素时会提示"UnChecked"警告，所以在访问list2中的元素时，如果访问不当就会在运行时引发ClassCastException
list2.add("123");

//这里尝试把"123"转化为Integer，将会引发ClassCastException
Integer num = (Integer) list2.get(1);
//这里访问正确
Object o = list2.get(1);//或者String num = (String)list2.get(1)
```

同时系统支持把没有泛型信息的变量赋值给具有泛型信息的变量，而不会提示任何警告或错误，如下：

```java
//类型形参为Integer的泛型
List<Integer> list = new ArrayList<>();
list.add(1);

//list的Integer类型信息被擦除
List list2 = list;

//把擦除后的list赋值给类型形参为String的泛型list3
List<String> list3 = list2;

//下面的访问将会引起ClassCastException
//等价于String num = 1；
String num = list3.get(0);
```

上述代码将会引发ClassCastException，因为list3的类型信息是String，编译器会把list3中的元素类型当作是String，而此时list3实际引用的变量泛型擦除后的list，泛型擦除后的list中的元素类型在编译时是Object，但在运行时却是Integer，所以在运行时，从list3中取出的元素是Integer类型，Integer是不可以强转成String的，从而引起ClassCastException。

所以总的来说，泛型擦除主要在以下两个方面：

- 1、编译之后，泛型会被擦除；(自动擦除)
- 2、把泛型变量赋值给原始变量时，泛型会被擦除。(手动擦除)

## 结语

本文简单整理了一下java语言的基本知识点，希望大家有所收获！

参考资料：

[java 中的 ==, equals 与 hashCode 的区别与联系](https://blog.csdn.net/justloveyou_/article/details/52464440)

[Java 自动装箱与拆箱的实现原理](https://www.jianshu.com/p/0ce2279c5691)

[为什么阿里不建议在for循环中使用"+"进行字符串拼接](https://mp.weixin.qq.com/s/Mvi-qq3-9Y_mi1eFKahEYw)