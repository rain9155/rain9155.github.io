---
title: java学习总结之集合框架
tags:
  - list
  - set
  - map
  - queue
  - 集合
categories: java
date: 2019-11-05 18:33:15
---


## 前言

在JDK1.2之前，java是没有完整的集合框架的，只有一些简单的可以扩展的**容器类**，如Vector、Stack、Hashtable等，这些容器类它们解决了数组不能动态扩容和使用复杂的问题，到了JDK1.2之后，为了管理这些容器类，就出现了集合框架这个概念，集合框架是为了表示和操作集合而规定的一种统一的标准的体系结构，它包含三大内容：对外的接口、接口的实现和对集合运算的算法（对某一种数据结构的算法），所以有了集合框架后，容器类一般改为叫**集合类**，常用的集合类有：Set、List、Map、Queue，下面会首先简单介绍集合框架的整体架构，然后分别介绍Set、List、Map、Queue的各自特点。

> 本文源码都是基于java7

## 一、Collection框架

图片来源[Java集合类的类图](https://blog.csdn.net/s20082043/article/details/39077095)

{% asset_img java1.png java %}

从这个类图中，可以看到各个集合类之间的继承关系，其中**Map没有继承Collection接口**，所以它独立出来，但是它也是集合框架的一部分，而且这里还有一个点没有画出来的是，Collection接口是**继承自Iterable**的，Iterable中有一个 iterator方法，它返回一个Iterator，可以用于遍历集合的元素，而Map是**没有继承自Iterable**的，但是它的各自实现类内部也实现了各自的Iterator，例如KeyIterator，ValueIterator和EntryIterator，通过特定方法返回，集合框架中的所有接口和类都在**java.util包**中，并且集合框架中所有的**具体类**都实现了Cloneable和Serialization接口，即它们的实例都是可以复制和可序列化的，有以下四种主要类型的集合：

* List(线性表)：存储一组有序的元素
* Set(规则集)：存储一组不重复的元素
* Map(映射表)：存储一组key-value映射，一个Map中不能包含相同的key，每个key只能映射一个
  value
* Queue(队列)：存储一组用先进先出方式处理的元素

下面分别讲解.

## 二、List

{% asset_img java2.png java %}

List是有序的Collection，使用此接口能够精确的控制每个元素插入的位置，用户能够使用索引来访问List中的元素，这类似于Java的数组，List允许有重复的元素，除了具有Collection接口必备的iterator()方法外，List还提供一个listIterator方法，返回一个ListIterator接口，和标准的Iterator接口相比，ListIterator多了一些add()之类的方法，允许添加，删除，设定元素，还能向前或向后遍历List，List常见的实现类有：**ArrayList，LinkedList，Stack和Vector**.

### 1、ArrayList

```java
transient Object[] elementData
```

它是一个大小可变的数组，在内存中分配连续的空间，它允许存储任意类型的对象，包括null，每个ArrayList实例都有一个容量（Capacity），即用于存储元素的数组的大小，这个容量可随着不断添加新元素而自动增加，ArrayList是一个线程不安全的类，ArrayList是集合框架出现之后用来取代Vector类的，两者的底层原理都是基于数组的算法，几乎一模一样。

> 在java7之前使用new ArrayList()创建一个List对象时，会初始化一个Capacity为10的Object数组，但是我并没有存储元素，就会造成空间的浪费；从java7之后，就开始优化了这个设计，使用new ArrayList()创建一个List对象时，底层只会初始化一个Capacity为0的空数组，只有在第一次调用add方法时，才会去初始化这个数组。

### 2、LinkedList

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    //...
}
```

它使用双向链表链表存储元素，它是单向队列、双向队列、List的实现类，所以它有着多种数据结构的实现，每一种数据结构的操作方法不同，它允许存储所有元素，包括null，提供了从两端提取、插入和删除元素的方法，LinkedList是一个线程不安全的类，和ArrayList不同，虽然它实现了List接口，但是它没有索引的概念，所以LinkedList不擅长查询操作。

> LinkedList类作为List接口的实现类，List中提供了根据索引查询元素的方法，如Object get(int index)，表示根据index位置获取对应的元素，但是LinkedList是链表，它没有索引的概念，所以LinkedList内部会采用遍历链表的方式获取到index位置的元素，该方法尽量少用，效率不高。

### 3、Vector

```java
protected Object[] elementData;
```

它是一个大小可变的数组，在内存中分配连续的空间，它允许存储任意类型的对象，包括null，Vector是一个线程安全的类，它的方法都有synchronized修饰，它的常用方法如下：

{% asset_img java3.png java %}

Vector非常类似ArrayList，由Vector创建的Iterator，虽然和ArrayList创建的Iterator是同一接口，但是，因为Vector是同步的，当一个Iterator被创建而且正在被当前线程使用，另一个线程改变了Vector的状态（例如，添加或删除了一些元素），这时调用Iterator的方法时将抛出ConcurrentModificationException，因此必须捕获该异常。

> ArrayList是集合框架出现之后用来取代Vector类的，ArrayList中的方法实现都是基于Vector的方法实现。

### 4、Stack

```java
class Stack<E> extends Vector<E>{
    //...
}
```

它继承自Vector，是java中栈的实现，它的存储特点是LIFO，即后进先出，它把数组的最后一个元素当作栈顶，它的常用方法如下：

{% asset_img java4.png java %}

> 官方建议：如果要使用栈，尽量使用ArrayDeque，它是Deque接口的实现类，表示双向队列，Deque接口提供了LIFO的堆栈操作和更完整的set，如Deque<Integer> stack = new ArrayDeque<>()；

### 5、ArrayList和Vector的区别与选择

**相同点：**

底层都是基于数组的算法，实现的逻辑大概一致，功能相同，在很多情况下可以互用。

**不同点：**

1、Vector线程安全，它的方法都用synchronized修饰，而ArrayList线程不安全，但是速度快；

2、但需要扩容时，Vector默认增长一倍，而ArrayList增长50%，有利于节约空间。

**如何选择？**

**ArrayList可以完全替代Vector**，因为它效率高且节约空间，同时在线程不安全的环境下，可以使用**List list = Collections.synchronizedList(new ArrayList())**来返回一个线程安全的ArrayList，所以在开发中，应该先考虑使用ArrayList。

### 6、ArrayList和LinkedList的区别与选择

**相同点：**

大家都实现了List接口，都是线程不安全的类。

**不同点：**

1、LinkedList底层数据结构是双向链表，而ArrayList底层数据结构是数组；

2、LinkedList底层采用的是链表结构的算法，所以它的插入和删除操作很快，而ArrayList底层采用的是数组结构的算法，所以它的查询和修改操作很快。

**如何选择？**

如果是插入和删除操作频繁，优先考虑LinkedList，如果是查询和修改操作频繁，优先考虑ArrayList，但在平时开发中，**使用ArrayList较多，根据开发环境来选择**。

## 三、Map

{% asset_img java5.png java %}

Map是以键-值存储元素的容器，根据关键字Key找到对应的数据Value，它常见的实现类有：**HashMap、TreeMap、HashTable、LinkedHashMap**.

### 1、HashTable

```java
private transient Entry<K,V>[] table;

private static class Entry<K,V> implements Map.Entry<K,V> {
    int hash;//Key的hashCode经过hash方法计算后的hash值
    final K key;
    V value;
    Entry<K,V> next;
    //...
}
```

HashTable采用了数组加链表的数据结构，能在查询和修改时分别继承数组和链表的优良特性，它是从java1就出现了，历史悠久，它是线程安全的哈希表，即它的put、get、remove等方法都加上了synchronized关键字，所以它的效率比较低，它不可以存储null键和值，它的初始容量initialCapacity 可以在构造函数时由用户指定，默认值为11，它里面有5个主要的成员，如下：

* count： 映射数量，Hashtable中Entry对象（映射）的个数;

* loadFactor：负载因子，在其容量自动增加之前可以达到多满的一种尺度，默认为0.75;

* threshold： 扩容阈值，对Hashtable进行扩容的阈值，等于initialCapacity * loadFactor;

* table[]: Entry数组，一个由Entry对象组成的链表数组，table数组的每一个数组成员就是一个链表；

* modCount：结构性修改次数， 记录Hashtable生命周期中结构性修改的次数，便于快速失败机制.


> 快速失败机制是指其在并发环境中进行迭代操作时，若其他线程对其进行了结构性的修改，这时迭代器能够立马感知到并且立即抛出ConcurrentModificationException异常，而不是等到迭代完成之后才告诉你。

当我们使用put(key, value)存储对象到HashTable中时，HashTable会先调用hash方法计算Key的hashCode，并返回新的hash值，然后通过与数组长度取模运算，定位到table数组中相应位置来储存Entry对象，如果该位置已经有元素了，即发生冲突，就调用equals() 比较Key，相同则替换旧的Value值，都不相同则创建新的Entry链入到该位置的链表中，即采用拉链法来解决冲突，在链入新的Entry前，会先检查数组是否达到threshold值，如果达到了，就需要resize，扩容（2倍 + 1）后重排。

> 如果使用到哈希表（HashMap、HashTable、HashSet等），作为key的对象要正确复写equals和hashCode方法。

### 2、HashMap

```java
transient Entry<K,V>[] table;

static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;<K,V> ；
     //...
}
```

HashMap和HashTable一样，底层都是基于hash的算法，都是数组加链表的数据结构，HashMap是非线程安全的，所以它效率高，它可以接收null键和值，HashMap中同样有**size(和count一样)、loadFactor、threshold、table[]和modCount**这5个重要的成员，含义都一样，HashMap的初始容量initialCapacity 也可以在构造函数时由用户指定，默认值为16，且HashMap的**大小必须为2^n**，如果你传入的initialCapacity 不是2^n，它会自动的替你取最接近initialCapacity 的2^n大小。

当你使用put(key, value)存储对象到HashMap中时，它的过程和HashTable的几乎一样，其中不同的有以下几点：

* 1、HashMap允许存储Key和Value为null的对象，如果Key为null，就会把这个映射放在数组的第一个位置，且只允许有一个Key为null的映射存在；
* 2、HashMap中用于定位映射在数组中的位置是通过&运算，而不是HashTable那样的%运算，所以HashMap的效率更高；
* 3、在HashMap的插入K/V对的过程中，总是先插入后检查是否需要扩容，而Hashtable则是先检查是否需要扩容后再插入，且HashMap的扩容大小是原来的2倍，而不是2倍+1；
* 4、HashMap的put操作是非线程安全的，而HashTable的是线程安全的。

至于get方法，大家可以自己分析并总结出它们的不同。

> 在java8之后，HashMap引入了红黑树，在单个hash值存储的元素个数大于8个时，就会把链表转换为红黑树，保证在最坏的情况下查询的效率是O(logn)，n是单个hash值存储的元素个数。

### 3、LinkedHashMap

```java
private transient Entry<K,V> header;
private final boolean accessOrder;//false表示按照插入顺序迭代，true表示按访问顺序迭代，默认为false

 private static class Entry<K,V> extends HashMap.Entry<K,V> {
        Entry<K,V> before, after;
     //...
 }
```

LinkedHashMap继承自HashMap，在此基础上，添加了**双向链表头结点header** 和  **标志位accessOrder** ，所以LinkedHashMap就是**HashMap + 双向链表**，它拥有HashMap的所有特性，同时额外维护了一个双向链表用于保持迭代顺序，HashMap中有一个init方法，会在构造函数执行完后调用，LinkedHashMap重写了该方法，完成了双向链表头结点的初始化，如下：

```java
//LinkedHashMap.java
@Override void init() {
    header = new Entry<>(-1, null, null, null);
    header.before = header.after = header;
}
```

我们发现这个双向链表是**循环链表**，通过header.after就可以拿到链表中的第一个结点，通过header.before就可以拿到链表中的最后一个结点。

所以LinkedHashMap是可以保持插入的顺序的（**当你调用put时，它会把插入的元素的放到链表的底部**），它还可以在构造时指定**accessOrder = true**来保持访问的顺序（**当你调用get时，它会把访问过元素的放到链表的底部**），这样当你在**迭代**LinkedHashMap时，它会从头到尾遍历双向链表，逐一输出双向链表的各个结点，这样就保持了LinkedHashMap的有序性，所以如果有人问你LinkedHashMap的有序性是怎样实现的，你就告诉他：**LinkedHashMap通过双向列表保证元素插入或访问的顺序，并重写了HashMap 的迭代器，当你迭代LinkedHashMap时，它会把其维护的双向链表进行迭代输出，这样就保证输出的元素是有序的**。

> 把accessOrder 置为true，就可以通过LinkedHashMap实现LRU算法 (Least recently used, 最近最少使用)，需要做到以下2个步骤：（假设当元素大于10个时就要删除最久没有被使用的元素）
>
> 1、编写一个类继承自LinkedHashMap，并重写LinkedHashMap的removeEldestEntry方法，返回size() > 10;
>
> 2、在构造这个类时，通过构造函数，指定accessOrder为true.
>
> 这样当你put或get时，它会把这个元素移动到链表的底部，从而保持链表的底部的元素是最近访问过的，而链表的头部的元素是最久没有被使用过的，当元素达到10以上时，LinkedHashMap自动会把链表头部的那个元素删除掉。

### 4、TreeMap

```java
private transient Entry<K,V> root = null;

static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left = null;
        Entry<K,V> right = null;
        Entry<K,V> parent;
        boolean color = BLACK;
    
    //...
}
```

TreeMap和前面3个Map不一样，并不是数组加链表的实现，而是基于红黑树实现的，所以TreeMap底层没有hash算法的实现，它的put、get、remove等都是基于红黑树的操作，所以Entry中就没有**hash**这个变量，取而代之的是left(左孩子结点)、right(右孩子结点)、parent(父亲结点)、color(是红色还是黑色结点)属性，红黑树是一颗**自平衡的二叉查找树**，它通过旋转和变色来保持树的平衡，保证在最坏的情况下查询的效率是O(logn)。

所以如果你想深入的了解TreeMap，你只要熟悉红黑树这种数据结构就行，推荐阅读[30张图带你彻底理解红黑树](https://www.jianshu.com/p/e136ec79235c)。

TreeMap和LinkedHashMap一样都是可以保证元素的有序性，但TreeMap并不是保证元素的插入顺序而是**保证Key的自然排序**，例如对于Key为对Integer来说，其自然排序就是数字的升序，对于Key为String来说，其自然排序就是按照字母表排序，那么它是如何保证的呢？

```java
private final Comparator<? super K> comparator;

//默认构造器，comparator为null
 public TreeMap() {
        comparator = null;
 }

//通过构造器指定TreeMap的comparator
 public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
 }

//以put方法举例
 public V put(K key, V value) {
     	//红黑树根节点
        Entry<K,V> t = root;
        if (t == null) {
           root = new Entry<>(key, value, null);
           //...
        }
        int cmp;
        Entry<K,V> parent;
     	//首先尝试获取Comparator
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                //如果Comparator不为null，就使用Comparator的compare方法来比较两个Key的大小
                cmp = cpr.compare(key, t.key);
                //...
            } while (t != null);
        }else {
            if (key == null)
                throw new NullPointerException();
            //如果Comparator为null, 就尝试把Key转成Comparable
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                //使用Comparable的compareTo方法来比较两个Key的大小
                cmp = k.compareTo(t.key);
                //...
            } while (t != null);
        }
        //...
        return null;
    }
```

我们发现，它的内部还有一个Comparator，可以通过构造函数指定，如果不指定默认为null，当我们添加元素时，它首先会尝试使用Comparator来比较两个元素的大小，如果Comparator为null，就尝试把Key转成Comparable，使用Comparable比较两个Key的大小，如果这个使用Key没有实现Comparable接口，就会报错，我们还发现**Comparator的优先级大于Comparable**，Comparable和Comparator都是用来**比较大小**的，关于它们的区别，推荐阅读[Comparable与Comparator浅析](https://blog.csdn.net/u013256816/article/details/50899416?utm_source=app)。

所以如果有人问你TreeMap的有序性是怎样实现的，你就告诉他：**TreeMap的有序性保证是通过使用Comparator或Comparable，把它保存的键值对根据Key排序，基于红黑树，从而保证TreeMap中所有键值对处于有序状态**。

> 对于Integer、Long等包装类和String类都实现了Comparable接口，所以以它们作为Key，TreeMap可以对它们进行自然排序，但是如果你的Key是**自定义类**，但是没有实现Comparable接口，在你插入元素时就会报错，所以如果你的Key是自定义类，你需要为TreeMap**指定Comparator或为你的自定义类实现Comparable接口**，自定义排序规则。

### 5、HashTable和HashMap的区别与选择

**相同点**：

底层都是基于hash算法实现，底层数据结构都是数组+链表。

**不同点**：

1、HashTable和HashMap的实现模板不一样，HashTable继承自陈旧的Dictionary抽象类，在java1.0引入，而HashMap是继承自AbstractMap抽象类，在java1.2引入，其中AbstractMap实现了Map接口，有很多Map的骨干实现；

2、HashTable是线程安全的，而HashMap是非线程安全的；

3、HashTable不允许存储为null的Key和Value，而HashMap允许存在一个为null的Key和多个为null的Value；

4、HashTable是把hash值通过对长度**取模**来定位映射在数组中的位置，而HashMap是通过与（长度-1）**相与**来定位映射在数组中的位置.

**如何选择**？

在**非并发环境**下，HashMap完全**可以替代**HashTable，因为HashMap是非线程安全的，所以它的效率比HashTable**更高**，而且它是把hash值通过与（长度-1）相与来定位映射在数组中的位置，所以它比HashTable的取模运算**更快**。

在**并发环境**下，HashMap是非线程安全的，这时可以用**ConcurrentHashMap**来替代HashTable，ConcurrentHashMap是HashMap在并发环境下的一个实现，它不像HashTable在读写数据时直接锁住整个数组，它采用**分段锁**，在读写数据时**只锁住你要读写的那一部分数据**，所以ConcurrentHashMap可以**支持多个线程**执行并发写操作及任意数量线程的读操作，所以并发效率远远**超过**HashTable。

综上所述，在并发环境下选择ConcurrentHashMap，在非并发环境下选择HashMap。

### 6、TreeMap和HashMap的区别与选择

**相同点**：

大家都继承自AbstractMap抽象类，大家的Value都可以为null的。

**不同点**:

1、HashMap底层是数组+链表，而TreeMap底层是红黑树;

2、HashMap中的元素是无序的，而TreeMap中所有的元素都按Key的自然排序;

3、HashMap中的Key可以为null，而TreeMap不可以。

**如何选择**：

如果你**对集合的顺序没有要求**，那么优先考虑HashMap，由于使用到hash算法，在HashMap中插入、删除和定位元素的平均效率是比TreeMap高的。

如果你**对集合的顺序有要求**，例如在迭代的情况下，我要求元素的输出是有序的，那么优先考虑LinkedHashMap或TreeMap，如果你希望你的元素**是按插入或访问顺序排序**的，你要选择LinkedHashMap；如果你对希望你的元素是按Key的自然顺序排序的，那么你要选择TreeMap。

综上所述，在没有顺序要求下，选择HashMap，因为TreeMap要保持元素的有序性，会导致效率比HashMap低，在有顺序要求下，看情况选择LinkedHashMap和TreeMap。

## 四、Set

{% asset_img java6.png java %}

Set是用来操作一组唯一、无序的对象，它最多有一个null元素，它有3个常用的实现类：

- HashSet：用来存储互不相同的任何元素.
- LinkedHashSet：继承自HashSet，使用链表扩展实现HashSet类，支持对元素的排序.
- TreeSet：可以确保所有元素是有序的.

其中LinkedHashSet继承自HashSet，LinkedHashSet的底层实现是LinkedHashMap，HashSet的底层实现是HashMap，TreeSet的底层实现是TreeMap，所以掌握了Map就等于掌握了Set.

### 1、HashSet

```java
private transient HashMap<E,Object> map;
private static final Object PRESENT = new Object();//map的Vaule，起到占位的作用

public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}


public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}


public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

//如果调用3个参数的构造函数，可以把HashSet的底层实现指定为LinkedHashMap
//注意它的访问修饰符不是public，所以我们不可以使用这个构造函数，这个构造函数是内部使用的，如LinkedHashSet
//这个dummy参数可以忽略，没有什么作用
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

我们可以看到，除了最后一个构造函数，其他所有的构造函数都是根据构造参数new了一个HashMap，至于最后一个3个参数的构造函数，我们无法调用，所以我们可以说HashSet的底层实现就是**HashMap**，我们知道HashMap中的Key是**唯一、可null、无序的**，所以HashSet就利用了这一个特性，当我们往HashSet中保存元素时，**这个元素就被作为Key**，如下：

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

所以我们**对HashSet中元素的操作就是对其内部map的Key的操作**，其实在HashMap中，Value的地位是比Key低的，Value只是作为Key的附属，有点男重女轻的思想，所以如果不需要建立映射关系，保存元素时，采用HashSet能有HashMap一样的效率。

### 2、LinkedHashSet

```java
public LinkedHashSet(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor, true);
}


public LinkedHashSet(int initialCapacity) {
    super(initialCapacity, .75f, true);
}


public LinkedHashSet() {
    super(16, .75f, true);
}

public LinkedHashSet(Collection<? extends E> c) {
    super(Math.max(2*c.size(), 11), .75f, true);
    addAll(c);
}
```

LinkedHashSet继承自HashSet，所以它的父类是HashSet，我们发现LinkedHashSet的构造函数都调用HashSet的带有**3个构造参数**的构造函数，在HashSet介绍中讲到，3个参数的构造函数中**会把HashSet的底层实现指定为LinkedHashMap**，所以LinkedHashSet的底层实现就是LinkedHashMap，而且LinkedHashSet类中只有这4个构造函数，**没有重写任何方法**，所以对LinkedHashSet操作就是对HashSet操作，只是HashSet的**底层实现由HashMap变成了LinkedHashMap**。

### 3、TreeSet

```java
private static final Object PRESENT = new Object();

TreeSet(NavigableMap<E,Object> m) {
    this.m = m;
}

public TreeSet() {
    this(new TreeMap<E,Object>());
}

public TreeSet(Comparator<? super E> comparator) {
    this(new TreeMap<>(comparator));
}

public TreeSet(Collection<? extends E> c) {
    this();
    addAll(c);
}

public TreeSet(SortedSet<E> s) {
    this(s.comparator());
    addAll(s);
}

```

从构造函数发现，TreeSet的底层实现都指定为**TreeMap**，和HashSet的逻辑一样，就不再累述了。

### 4、HashSet、LinkedHashSet和TreeSet的选择

区别就不讲了，和Map家族的区别差不多，这里讲一下它们的使用场景，如果不需要维护元素被插入的顺序，就应该使用HashSet，更加高效，因为HashSet底层使用hash算法来定位元素，如果产生的冲突少的话，它的效率可以达到O(1)常数级别，而TreeSet因为要保持有序性，所以就要进行比较等额外操作，它的时间复杂度为O(logn)；

如果需要保持元素的有序性，看情况选择LinkedHashSet（插入或访问有序）和TreeSet（Key自然排序或你自定义排序规则）。

## 五、Queue

{% asset_img java7.png java %}

Queue通常用于操作存储一组队列方式的对象，它的特点是先进先出（FIFO），Deque继承自Queue，是双端队列的简称(double-ended queue)，支持在两端插入和删除元素，在Deque接口增加的方法有：addFirst(e)、removeFirst(e)、addLast(e)、removeLast(e)、getFirst()和getLast()等，我们常用的有LinkedList、ArrayDeque，这里就简单介绍一下：

* LinkedList：底层是双向链表的实现，在添加和删除元素时比ArrayList具有更好的性能，在查询和更新元素方便弱于ArrayList，如果数据量都不大，两者的性能差不多，LinkedList作为队列使用时，**尽量避免Collection的add()和remove()方法，而是要使用offer()来加入元素，使用poll()来获取并移出元素**，它们的优点是通过返回值可以判断成功与否，add()和remove()方法在失败的时候会抛出异常。
* ArrayDeque：底层是数组的实现，这个数组是**循环数组**，因为要满足在数组两端插入或删除元素的需求，它可以作为队列、双端队列、栈来使用，它是官方推荐用来**代替Stack**的，它的默认容量为16，容量必须为2的幂次方（和HashMap的一样，因为它的底层也是通过位运算来定位元素，2^n更方便完成一些位运算的骚操作），它的性能比LinkedList还好。

以上是非阻塞队列的两个实现，在java中，还有阻塞队列这一说，它在java5中加入，使用阻塞队列更加方便的实现**生产者-消费者**，阻塞队列经常用于多线程环境，如线程池中，关于阻塞队列的介绍可以看[java线程池](https://rain9155.github.io/2019/07/19/java线程池/)，阻塞队列都继承自BlockingQueue接口 ，而BlockingQueue 继承自Queue接口。

## 六、集合的遍历方式

集合的遍历大同小异，可以分别Collection家族的遍历和Map家族的遍历.

### 1、Collection的遍历

以List为例，对于**List**来说，有3种遍历方式，分别是：

* 通过for循环遍历
* 使用迭代器遍历(对于List来说，可以使用Iterator或ListIterator)
* 通过foreach循环遍历（语法糖，反编译后还是通过迭代器来遍历）

```java
List<Integer> list = new ArrayList<>(){{
    add(1); add(2); add(3); add(4);
}};

//1、通过for循环遍历
for(int i = 0; i < list.size(); i++){
    System.out.print(list.get(i) + " ");
}
System.out.println();

//2、使用迭代器遍历
Iterator<Integer> listIterator = list.iterator();
while (listIterator.hasNext()){//hasNext()：判断当前指针后是否有下一个元素
    System.out.print(listIterator.next() + " ");//next()：移动指针，获取下一个元素
}
System.out.println();

//3、通过foreach循环遍历
for(Integer num : list){
    System.out.print(num + " ");

}
```

对于**Set**来说，它不能通过for循环遍历，它只能使用**迭代器和foreach遍历**，因为它没有索引的概念，对于**Queue**来说，它能通过**循环+poll方法、迭代器和foreach遍历**，如果要遍历**List**集合，对于ArrayList、Vector来说，使用for循环的效率更高，对于LinkedList来说，使用迭代器的效率更高。

### 2、Map的遍历

以HashMap为例，对于**HashMap**来说，有2种遍历方式，分别是：

* 通过Map的ketSet方法返回KeySet，遍历KeySet，通过Key取出Value（二次取值）
* 通过Map的entrySet方法返回EntrySet，遍历EntrySet，取出Key和Value（Map数量量大时，推荐使用本方法遍历Map）

不管是KeySet还是EntrySet，它们都是Set集合，Set集合可以通过**迭代器和foreach遍历**，下面示例使用foreach遍历：

```java
Map<Integer, String> map = new HashMap<>(){{
    put(1, "1"); put(2, "2"); put(3, "3"); put(4, "4");

}};

//1、遍历KeySet，通过Key取出Value
Set<Integer> keySet = map.keySet();
for(Integer key : keySet){
    System.out.print(key + "--" + map.get(key) + " ");
}
System.out.println();

//2、遍历EntrySet，取出Key和Value
Set<Map.Entry<Integer, String>> entrySet = map.entrySet();
for(Map.Entry<Integer, String> entry : entrySet){
    System.out.print(entry.getKey() + "--" + entry.getValue() + " ");
}
```

foreach遍历底层其实还是通过迭代器遍历。

## 总结

其实还有两个很少用到，但在特殊场景却一定会用到它的两个集合没有讲到，分别是：PriorityQueue(优先级队列)、WeakHashMap(Key为弱引用的HashMap)，PriorityQueue可以指定比较器实现**小顶堆和大顶堆**，WeakHashMap可以用在**内存有限**的环境下，防止OOM，关于它们的具体使用可以自行查阅资料。

本文主要简单介绍了集合框架中经常用到的集合类：ArrayList、LinkedList、HashMap、LinkedHashMap、TreeMap、HashSet、LinkedHashSet、TreeSet、ArrayDeque，和一些古老的容器类：Stack、Vector、HashTable，其中容器类已经不推荐使用了，它们都有各自的替代品，分别是：ArrayDeque、ArrayList、HashMap，本文还讲解了集合类之间各自的区别和使用场景，还有集合的迭代方式，在使用集合时，要善用**Collections和Arrays**工具类，它里面有很多对集合操作的工具方法，能在开发中简化我们的工作量。

以上就是本文的全部内容，如有错误，欢迎指出！

参考资料：

[彻头彻尾理解 LinkedHashMap](https://blog.csdn.net/justloveyou_/article/details/71713781)

[彻头彻尾理解 HashTable](https://blog.csdn.net/justloveyou_/article/details/72862373)

[Java集合类详解](https://blog.csdn.net/softwave/article/details/4166598)

[Java ArrayDeque源码剖析](https://www.cnblogs.com/CarpenterLee/p/5468803.html)

[Java中modCount的作用](https://blog.csdn.net/badguy_gao/article/details/78989637)