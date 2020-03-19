---
title: java学习总结之反射
tags: 反射
categories: java
date: 2020-03-19 14:59:32
---


## 前言

在java中，反射就是在程序**运行时**动态的获取某一个类的元数据(metadata，描述数据的数据)的过程，这些元数据包括构造器、方法、成员变量、内部类、接口、父类等，通过反射，我们可以在程序**运行时**动态地去操作类的方法、成员变量等信息，所以，在java中，反射为我们提供了一种**动态访问、修改**类的能力，掌握反射，对我们加深java语言的理解很有帮助，反射大部分所使用到的类都在**java.lang.reflect**包下。

## 为什么使用反射

首先我们要知道在java中，对象分为**编译时类型**和**运行时类型**，例如：**Object obj = new String("123")；** 其中obj变量的编译时类型为Object类型，运行时类型为String类型，编译时类型在**编译期**就确定了变量的类型，而运行时类型则在**运行期**才确定变量的类型。

这时假设我需要通过obj 变量调用String中的length()方法，当我写下**obj.length()**这句代码时，编译就会报错，因为编译时会检查obj变量的编译时类型Object，发现它不存在length()方法，编译不通过，报错，这时的解决方法是：把obj强转成String，然后再调用length()方法，如：**((String)obj).length()；**这时编译就通过了。

但是如果此时我并不知道obj变量的运行时类型是String，我怎么强转，我怎么调用它的length()方法？这时就要通过反射来实现了，例如下面的场景：

```java
//外部调用getLen方法，把obj变量传入方法
public static void main(String[] args){
    Object obj = new String("123");
    System.out.println(getLen(obj));
}

//在方法内部的我们只知道obj的编译时类型为Object，并不知道它的运行时类型是什么？
public int getLen(Object obj){
    //通过反射调用obj的length()方法
    Method method = obj.getClass().getMethod("length");
    return (Integer) method.invoke(obj);
}
```

上述我们在执行一个getLen方法，方法传入了一个Object类型的对象，但是我们却需要调用对象运行时类型的方法length()，我们此时并不知道该对象的运行时类型，只知道对象编译时类型是Object，这时就需要通过反射来实现该方法的调用，obj变量在运行时是String类型，所以上述代码在运行时通过反射调用了obj的length()方法。

所以为什么要使用反射呢？这是因为有时候**我们在编译时无法预知对象的运行时类型，但却需要使用对象的运行时类型的信息**，这时就需要通过反射来实现。

## 反射的基本使用

### 1、获取Class对象

每个类被加载进JVM时，JVM会为这个类创建一个java.lang.Class对象，Class对象就是类的元数据，通过这个Class对象，我们可以在代码运行时访问到这个类的所有信息，Class对象是代码运行时访问类信息的唯一入口，所以要想使用反射，我们首先要获取到类的Class对象，获取Class对象主要有以下3种方法：

- 1、**使用Class的forName(String className)方法**：该方法需要传入类的全限定名，通过类的全限定名加载这个类进入JVM，并返回该类的Class对象，如果无法加载这个类就抛出一个ClassNotFoundException异常；
- 2、**调用某个类的class属性**：每个类都有一个class属性，调用类.class会返回该类的Class对象；
- 3、**调用某个对象的getClass方法**：getClass方法是Object类中的一个方法，所以java中的所有对象都可以调用这个方法，getClass方法会返回该对象所属类的Class对象；

下面使用上面的3种方法获取String类的Class对象，如下：

```java
public static void main(String[] args) throws ClassNotFoundException {
    //1、使用Class的forName(String className)方法
    Class<?> stringClass1 = Class.forName("java.lang.String");
    //2、调用String类的class属性
    Class<String> stringClass2 = String.class;
    //3、调用String对象的getClass方法
    Class<?> stringClass3 = new String().getClass();
}
```

在JDK5之后，Class添加了**泛型支持**，允许使用泛型来限制Class的类型，例如上面的String.class，它的Class类型为Class<String\>，它表示Class对应的类为String类，如果Class对应的类未知，可以使用Class<?>或Class表示，在java中，通过forName或getClass方法获取的Class对象的对应的类是**未知的**，而通过某个类的class属性返回的Class对象的对应的类是**确定的**，通过泛型限制，可以避免使用反射创建对象时进行强制类型转换。

在java中，除了类有对应的Class对象外，**8大基本数据类型+void类型**都有自己的Class对象，但是这些基本类型并不是对象，所以它们不能通过getClass方法获取Class对象，同时基本类型也没有类名的概念，所以它们也不能通过forName方法获取Class对象，它们只能通过XX.class的方式获取相应的Class对象，如：int.class、byte.class、short.class、long.class、double.class、float.class、char.class、boolean.class和void.class，JVM中会内置这九大类型的Class对象，无需我们从外部加载，同时在这九大类型的对应的包装类中都有一个**TYPE**常量，这个TYPE常量就是对应基本类的Class对象，但是包装类的Class对象并不等于基本类的Class对象，以int为例，如下：

```java
public static void main(String[] args) throws ClassNotFoundException {
    System.out.println(int.class == Integer.TYPE);//输出true
    System.out.println(int.class == Integer.class);//输出false
}
```

在java中，数组也是一个引用类型，也是一个对象，所以数组也有自己的Class对象，数组可以通过getClass方法和XX.class方式获取自己的Class对象，以int[]为例，如下：

```java
public static void main(String[] args) throws ClassNotFoundException {
    int[] ints = {};
    //1、通过XX.class方式获取数组的Class对象
    Class<int[]> intsClass1 = int[].class;
    //2、通过getClass方法获取数组的Class对象
    Class<?> intsClass2 = ints.getClass();
}
```

### 2、通过Class对象获取类信息

一旦获取到类的Class对象后，就可以通过Class对象获取类中的所有信息，这些信息包括：构造器、方法、成员变量、内部类、外部类、接口、父类、注解、类的修饰符，类名、类所在包等，但是我们常用的类信息只有**构造器、方法、成员变量**，它们的元数据代表如下：

- **Constructor**：代表类的构造器对象；
- **Method**：代表类的方法对象；
- **Field**：代表类的成员变量对象；

在Class类中都有这些元数据的相应的获取方法，通过Class对象获取这些元数据的相应方法的名称命名都有一个规律：如果获取方法中**不带Declared**关键字的，那么这个方法只能获取到类的**public**属性，**包括继承的**；而如果这个获取方法**带Declared**关键字，那么这个方法就能获取到类的所有**public、protected、default、private**属性，但**不包括继承的**，而子类是无法继承父类的构造器的，所以通过子类获取构造器时，是无法获取到父类的构造器，只能获取到属于子类的构造器，而对于方法、成员变量它们是可以继承自父类的。

下面分别介绍它们的获取方法和使用场景：

#### 2.1、获取类的Constructor

下面的4个方法可以获取Class对应类的构造器对象Constructor：

- 1、**Constructor<T\> getConstructor(Class<?>... parameterTypes)**：返回此Class对应类的、带指定形参列表的**public**构造器对象，传入的形参也要用对应类型的Class对象表示；
- 2、**Constructor<?>[] getConstructors()**：以数组的形式返回此Class对应类的所有**public**构造器对象；
- 3、 **Constructor<T\> getDeclaredConstructor(Class<?>... parameterTypes) **：返回此Class对应类的、带指定形参列表的构造器对象，与访问权限无关，传入的形参也要用对应类型的Class对象表示；
- 4、**Constructor<?>[] getDeclaredConstructors()**：以数组的形式返回此Class对应类的所有构造器对象，与访问权限无关.

获得到构造器对象Constructor后，就可以调用Constructor的**newInstance(Object ... initargs)方法**创建对应类的实例，newInstance方法的initargs参数就是构造器中需要传入的参数，它是一个可变参数，如下我通过反射创建了一个值为123的String对象：

```java
//通过反射创建对象
public static void main(String[] args) throws IllegalAccessException, InvocationTargetException, InstantiationException, NoSuchMethodException {
    //1、获取String类的Class对象
    Class<String> stringClass = String.class;
    //2、通过Class对象获取一个带String类型参数的构造器Constructor
    Constructor<String> stringConstructor = stringClass.getConstructor(String.class);
    //3、使用Constructor的newInstance方法创建String对象，并传入该构造器需要的参数“123”
    String string = stringConstructor.newInstance("123");
    //输出：123
    System.out.println(string);
}
```

如果你需要调用类的默认构造器来创建对象，这时调用Constructor的newInstance方法时就不需要传入参数，类的默认构造器就是没有参数的那个构造器，同时为了简化获取Constructor这一步骤，java允许我们直接使用**Class对象的newInstance()**方法来调用类的**默认构造器**来创建对象，如下我通过Class对象的newInstance方法创建了一个空String对象：

```java
//通过反射创建对象
public static void main(String[] args) throws IllegalAccessException, InstantiationException{
    //1、获取String类的Class对象
    Class<String> stringClass = String.class;
    //2、调用String类的默认构造器来创建对象
    String string = stringClass.newInstance();
    System.out.println(string);
}
```

> Class对象的newInstance方法从java9之后就被标记为Deprecated，java9之后推荐使用Constructor的newInstance方法来反射创建对象实例。

为什么要使用Constructor来创建对象呢，直接使用new不行吗？这是因为某些场景下我们无法直接调用某个类的构造器，例如在一些框架中，传给我们的都是关于类的字符串(类的全限定名)，我们只能通过Class的forName方法获取到Class对象，然后获取到Constructor对象，然后通过Constructor对象来创建实例。

#### 2.2、获取类的Method

下面的4个方法可以获取Class对应类的方法对象Method：

- 1、**Method getMethod(String name, Class<?>... parameterTypes) **：返回此Class对应类的、指定名称的、带指定形参列表的**public**方法对象，传入的形参也要用对应类型的Class对象表示；
- 2、**Method[] getMethods() **：以数组的形式返回此Class对应类的所有**public**方法对象；
- 3、**Method getDeclaredMethod(String name, Class<?>... parameterTypes) **：返回此Class对应类的、**非继承的**、指定名称的、带指定形参列表的方法对象，与访问权限无关，传入的形参也要用对应类型的Class对象表示；
- 4、 **Method[] getDeclaredMethods() **：以数组的形式返回此Class对应类的、**非继承的**所有方法对象，与访问权限无关.

获取到方法对象Method后，就可以通过Method的**invoke(Object obj, Object... args)**方法来调用Method对应的方法，其中invoke方法的返回值就是被调用方法的返回值，而invoke方法的obj参数表示被调用方法的所属对象，args参数表示被调用方法的需要传入的参数，如下我通过反射调用String对象的length方法：

```java
//通过反射调用方法
public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException {
     //1、获取String类的Class对象
    Class<String> stringClass = String.class;
    //2、通过Class对象获取名称为length、没有参数的方法对象method
    Method method = stringClass.getMethod("length");
    //3、通过invoke方法执行String对象的length方法，返回String的长度
    Integer len = (Integer) method.invoke(stringClass.newInstance());
    //输出：0
    System.out.println(len);
}
```

如果需要调用的是**静态方法**，那么invoke方法的第一个参数obj参数就要**传入null**，因为静态方法不属于任何对象，静态方法只属于对应的类，如下我通过反射调用String的valueOf(int)静态方法：

```java
//通过反射调用静态方法
public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException {
    //1、获取String类的Class对象
    Class<String> stringClass = String.class;
    //2、通过Class对象获取名称为valueOf、参数类型为int的方法对象method
    Method method = stringClass.getMethod("valueOf", int.class);
    //3、通过invoke方法执行String类的valueOf方法，返回字符串“123”
    //注意：这里第一个参数传入了null，而不是String实例
    String num = (String) method.invoke(null, 123);
    //输出：123
    System.out.println(num);
}
```

如果需要调用的方法有一个**数组类型或可变参数类型**的参数，那么调用时就需要注意，而可变参数底层实现其实就是数组，下面我以数组为例，如下：

```java
public class Main {
    
    //通过反射调用参数含有数组类型的方法
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        //获取Main类的Class对象
        Class<Main> mainClass = Main.class;

        //情况1：数组的元素类型为基本数据类型
        Method method1 = mainClass.getMethod("doWork", int[].class);
        //method1.invoke(null, 1, 2, 3);//错误
        method1.invoke(null, new int[]{1, 2, 3});//正确
        method1.invoke(null, new Object[]{new int[]{1, 2, 3}});//正确

        //情况2：数组的元素类型为引用类型
        Method method2 = mainClass.getMethod("doWork", String[].class);
        //method2.invoke(null, "1", "2", "3");//错误
        //method2.invoke(null, new String[]{"1", "2", "3"});//错误
        method2.invoke(null, new Object[]{new String[]{"1", "2", "3"}});//正确
    }
    
    //数组的元素类型为基本数据类型
    public static void doWork(int[] nums){
        for(int num : nums){
            System.out.print(num + " ");
        }
    }

    //数组的元素类型为引用类型
    public static void doWork(String[] strs){
        for(String str : strs){
            System.out.print(str + " ");
        }
    }
}
```

可以看到反射调用的方法的参数含有数组类型，在执行invoke方法时传参会很容易出错，所以为了保险起见，建议以后反射调用参数含有数组类型的方法时，**把被调用方法的实际参数统统作为Object数组的元素即可，如：Method对象.invoke(方法所属对象，new Object[]{参数1，参数2，...})**。

#### 2.3、获取类的Field

下面的4个方法可以获取Class对应类的成员变量对象Field：

- 1、**Field getField(String name) **：返回此Class对应类的、指定名称的**public**成员变量对象；
- 2、**Field[] getFields() **：以数组的形式返回此Class对应类的所有**public**成员变量对象；
- 3、**Field getDeclaredField(String name) **：返回此Class对应类的、**非继承的**、指定名称的成员变量对象，与访问权限无关；
- 4、**Field[] getDeclaredFields() **：以数组的形式返回此Class对应类的、**非继承的**所有成员变量对象，与访问权限无关.

获得成员变量对象Field后，就可以通过Field的**setXX(Object obj)**方法和**getXX(Object obj)**方法为成员变量**赋值**和向成员变量**取值**，其中setXX或getXX方法中的obj参数表示成员变量所属的对象实例，而XX对应8种基本数据类型，如果成员变量为引用类型，则取消setXX或getXX方法中XX，例如下面我通过反射为一个引用类型变量和一个基本数据类型变量赋值和取值：

```java
public class Main {

    //引用类型
    public String str = "123";
    //基本数据类型
    public int num = 123;

     //通过反射访问成员变量
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        Main mainInstance = new Main();
        //1、获取Main类的Class对象
        Class<Main> mainClass = Main.class;
        //2、通过Class对象分别获取名为str和名为num的成员变量对象Field
        Field field1 = mainClass.getField("str");
        Field field2 = mainClass.getField("num");
        //3、分别通过Field的setXX方法赋值，getXX方法取值
        //重新赋值，如果是引用类型就直接调用set方法就行，如果是基本数据类型就可以调用setXX方法
        field1.set(mainInstance, "321");
        field2.setInt(mainInstance, 321);
        //取值，如果是引用类型就直接调用get方法就行，如果是基本数据类型就可以调用getXX方法
        String str = (String) field1.get(mainInstance);
        int num = (Integer)field2.getInt(mainInstance);
        //输出
        System.out.println(str);//输出：321
        System.out.println(num);//输出：321
    }
    
}
```

如果成员变量为静态成员变量，那么调用setXX或getXX方法时第一个参数就直接传**null**，如果这个成员变量是一个**私有变量**，那么还要调用**setAccessible(true)**方法后才能访问这个变量，如下我通过反射访问了一个名为data的私有变量：

```java
public class Main {

    //私有成员变量
    private int data = 123;

    //通过反射访问私有成员变量
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        Main mainInstance = new Main();
        //1、获取Main类的Class对象
        Class<Main> mainClass = Main.class;
        //2、通过Class对象获取名为data的成员变量对象Field，这里使用了getDeclaredField方法，因为getDeclaredField方法可以获取到私有变量
        Field field = mainClass.getDeclaredField("data");
        //注意：这里调用了setAccessible(true)方法把私有成员变量设置为可访问
        field.setAccessible(true);
        //3、通过Field的getXX方法取值
        int data = field.getInt(mainClass.newInstance());
        //输出：123
        System.out.println(data);
    }
}

```

对变量调用了setAccessible(true)后，就会在使用时关闭访问权限的检查，该变量就不会受访问权限的控制，我们就可以任意的修改和访问它。

其实不止Field可以调用setAccessible方法，对于Constructor、Method等也可以调用setAccessible方法，因为setAccessible方法是在**AccessibleObject**类中，而Field、Constructor、Method都继承自AccessibleObject，所以当以后想要通过Class对象访问**private**的Field、Constructor、Method时，都需要先调用**setAccessible(true)**关闭访问权限的检查后才能访问。

### 3、小结

以上就是反射的基本使用，下面再总结一下：

通过反射创建对象，需要通过下面三个步骤：

- 1、获取该类的Class对象；
- 2、利用Class对象的getConstructor或getDeclaredConstructor方法来获取指定的构造器对象Constructor；
- 3、调用Constructor的newInstance方法来创建相应的对象.

通过反射调用方法，需要通过下面三个步骤：

- 1、获取该类的Class对象；
- 2、利用Class对象的getMethod或getDeclaredMethod方法来获取指定的方法对象Method;
- 3、调用Method的invoke方法来执行相应的方法.

通过反射访问成员变量，需要通过下面三个步骤：

- 1、获取该类的Class对象；
- 2、利用Class对象的getField或getDeclaredField方法来获取指定的成员变量对象Field;
- 3、调用Field的setXX和getXX方法来访问相应的成员变量。

如果通过Class对象获取到的属性是私有的，还要调用setAccessible(true)方法关闭访问权限的检查。

## 通过反射获取变量的类型

在JDK5之前，java中只有原始类型(raw types)和基本类型(primitive types)，它们都用Class类表示，从JDK5开始引入了泛型(ParameterizedType)，这时为了统一表示这些类型就引入了**Type**接口，同时为了扩展java的泛型，还引入了数组类型(GenericArrayType)、类型变量(TypeVariable)、通配符类型(WildcardType)，这5种类型的父接口都是Type，他们的各自含义如下：

- **Class**：原始类型或基本类型，原始类型表示我们平常所使用的类、注解、枚举、数组等，基本类型表示基本数据类型，如int、byte、short等，它们都用Class类表示；
- **ParameterizedType**：参数化类型，表示java5引入的泛型类型，如List<String\>、Map<String, Integer>等；
- **GenericArrayType**：数组类型，表示带有泛型的数组，如T[]、List<String\>[]等；
- **TypeVariable**：类型变量，表示泛型的尖括号中的类型，如T、K、V等；
- **WildcardType**：通配符类型，表示泛型尖括号中带有上限或下限的通配符表达式，如? extends String、? super String等.

泛型经过编译后会被自动擦除，所以最终JVM中运行的都是Class文件，Class类是反射的基础。

当一个成员变量被定义在类中的时候，它的编译时类型就已经确定，当我们通过Class对象获取到这个成员变量的Field对象后，就可以通过Field的**getType**方法返回它的编译时类型，即这个成员变量的数据类型，getType方法会返回一个Class对象，代表这个数据类型的Class对象，如下：

```java
public class Main {

    public String str = "123";

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
        Class<Main> mainClass = Main.class;
        Field field = mainClass.getField("str");
        //调用Field的getType方法返回Field对应变量的数据类型
        Class<?> type = field.getType();
        //输出：java.lang.String
        System.out.println(type.getName());
    }
}
```

但是如果这个成员变量的类型是一个泛型，那么Field的getType方法返回的Class对象就会丢失了泛型尖括号中的类型参数，例如：Map<Integer, String> map = new HashMap<>()；获取到map变量的Field对象后，调用getType方法返回的Class类就是java.util.Map，丢失了尖括号中的类型参数信息，所以如果成员变量的类型是一个泛型，你可以通过Field的**getGenericType**方法返回它的参数化类型ParameterizedType，通过ParameterizedType就可以获取到泛型信息，如下：

```java
public class Main {

    public Map<Integer, String> map = new HashMap<>();

    public static void main(String[] args) throws NoSuchFieldException{
        Class<Main> mainClass = Main.class;
        Field field = mainClass.getField("map");
        //调用Field的getGenericType方法返回Field对应变量的数据类型
        //注意：getGenericType方法的返回值是一个Type类型，这说明它可以向下转型为Class、ParameterizedType、GenericArrayType、TypeVariable、WildcardType中的一个
        Type type = field.getGenericType();
        //我们已经知道它是一个泛型
        if (type instanceof ParameterizedType) {
            //强转为参数化类型ParameterizedType
            ParameterizedType parameterizedType = (ParameterizedType) type;
            //通过ParameterizedType的getRawType方法返回它的原始类型
            Type rawType = parameterizedType.getRawType();
            //通过ParameterizedType的getActualTypeArguments方法返回泛型中的类型信息
            Type[] types = parameterizedType.getActualTypeArguments();
            System.out.println("原始类型：" + rawType.getTypeName());
            for(Type t : types){
                System.out.println("类型形参： " + t.getTypeName());
            }
            //输出：
            //原始类型：java.util.Map
            //类型形参： java.lang.Integer
            //类型形参： java.lang.String
        }
    }
}
```

可以看到，如果成员变量的类型是一个泛型，就可以通过ParameterizedType的**getRawType**方法返回它的没有泛型信息的原始类型，通过ParameterizedType的**getActualTypeArguments**方法返回它的泛型信息，getActualTypeArguments方法它的返回值是一个Type数组，因为泛型的<>中会有多个Type，所以如果泛型的<>中某个Type是一个类型变量T，那么就可以把这个Type转成**TypeVariable**，通过TypeVariable的相应方法获取更多关于T的信息，如果<>中某个Type是一个通配符类型? extends XX或？super XX，那么就可以把这个Type转成**WildcardType**，通过WildcardType的相应方法获取更多关于通配符的信息，**GenericArrayType**同理。

所以对于数据类型为基本类型或原始类型的成员变量来说，通过getType方法就可以获取它的相应类型的Class对象；对于数据类型含有泛型的成员变量来说，就可以通过getGenericType方法获取它的相应类型，它返回一个Type，这个Type可以向下转型为ParameterizedType、GenericArrayType、TypeVariable中的一个，通过实际类型的相应方法获取泛型信息。

## 结语

本文通过对Class、Constructor、Method、Field、Type的基本使用来简单的介绍了一下java的反射，正是由于反射的存在，才使得java具有动态性、灵活性，反射在一些java的框架上应用非常广泛，如Spring等，对于我们平常开发的应用，主要就是本文的内容，同时反射也会和一些设计模式结合使用，让设计模式具有更好的扩展性，具体可以查看以下两篇文章:

[工厂方法模式](https://blog.csdn.net/Rain_9155/article/details/82942275)

[静态和动态代理模式](https://juejin.im/post/5db2fbd0518825645a5ba18b)

但同时反射也有它的缺点，反射的缺点就是执行效率低，因为反射需要按名称来检索类和方法，有一定的时间开销，同时反射会很容易破坏类的原本结构，所以在一些不必要的场景下不要频繁的使用反射。

以上就是本文的全部内容，希望大家有所收获！

参考资料：

[java中的Type](https://www.cnblogs.com/linghu-java/p/8067886.html)

[了解神秘的Java反射机制](https://blog.csdn.net/carson_ho/article/details/80921333)







