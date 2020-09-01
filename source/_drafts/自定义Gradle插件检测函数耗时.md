---
title: 自定义Gradle插件检测函数耗时
tags:
- gradle
- asm
- aop
Categories: grade
---

## 前言

- 上一篇文章：[Gradle的快速入门学习](https://rain9155.github.io/2020/06/26/Gradle的快速入门学习/)

上一篇文章讲解了Gralde的入门知识，其中讲到了如何自定义Gralde插件，本文就通过**Asm**和**Transfrom**来自定义一个简单的Gradle插件，这个Gradle插件它可以统计方法的耗时，并当方法的耗时超过阀值时，通过Log打印在控制台上，然后我们通过Log可以定位到耗时方法的位置，帮助我们找出耗时方法，一个很简单的功能，原理也很简单，这其中需要使用到Asm知识和Transfrom知识，所以本文首先会介绍Asm和Transfrom相关知识点，最后再介绍如何使用Asm和Transform来实现这个Gradle插件，如果你对Asm和Transfrom已经很熟悉了，可以跳过这两节。

> 源码位置在文末

## Asm

官方地址：[ASM](https://asm.ow2.io/)

官方教程：[ASM4-guide(英文版)](https://asm.ow2.io/asm4-guide.pdf)、[ASM4-guide(中文版)](https://github.com/rain9155/PluginDemo/blob/master/doc/ASM4-guide(中文版).pdf)

Asm是一个通用的Java字节码操作和分析框架, 它提供了一些简单易用的字节码操作方法，可以直接以二进制的形式修改现有类或动态生成类，简单地来说，Asm就是一个**字节码操作框架**，通过Asm，我们可以凭空生成一个类，或者修改现有的类，Asm相比其他的字节码操作框架如Javasist、AspectJ等的优点就是体积小、性能好、效率高，但它的缺点就是学习成本高，不过现在已经有IntelliJ插件[ASM Bytecode Outline](https://plugins.jetbrains.com/auth#access_token=1597742367104.6ee795af-5472-40b9-b586-e882ffbff43d.7334cf3f-10b4-47a4-9e77-07e382775999.0-0-0-0-0%3B1.MC0CFQCMwSIpa4MbttWs1X4KLFJ%2BhHK7IAIUDDKsMI4XaK2gN2DfsT21DYjWgGc%3D&token_type=Bearer&expires_in=3600&scope=0-0-0-0-0&state=f0b84383-ffef-42c2-81a4-d201d8cdf52f)可以替我们自动的生成Asm代码，所以对于想要入门Asm的人来说，它还是很简单的，我们只需要简单的学习一下Asm的相关api的含义，在此之前希望你已经对JVM的基础知识：类型描述符、方法描述符、Class文件结构有一定的了解。

Asm中有两类api，一种是基于树模型的tree api，一种是基于访问者模式的visitor api，其中visitor api是Asm最核心和基本的api，所以对于入门者，我们需要知道visitor api的使用，在visitor api中有三个主要的类用于**读取、访问和生成**class字节码：

- **ClassVisitor**： 它是用于**访问**calss字节码，它里面有很多visitXX方法，每调用一个visitXX方法，就表示你在**访问**class文件的某个结构，如Method、Field、Annotation等，我们通常会扩展ClassVisitor，利用[代理模式](https://juejin.im/post/6844903978342301709)，把扩展的ClassVisitor的每一个visitXX方法的调用委托给另外一个ClassVisitor，在委托的前后我们可以添加自己的逻辑从而达到**转换**、**修改**这个类的class字节码的目的；

- **ClassReader**：它用于**读取**以字节数组形式给出的class字节码，它有一个**accept**方法，用于接收一个ClassVisitor实例，**accept**方法内部会调用ClassVisitor的visitXX方法来访问已**读取**的class文件；

- **ClassWriter**：它继承自ClassVisitor，可以以二进制形式**生成**class字节码，它有一个**toByteArray**方法，可以把已**生成**的二进制形式的class字节码转换成字节数组形式返回.

  ClassVisitor、ClassReader、ClassWriter这三个之间一般都是需要组合使用的，下面通过一些实际的例子快速掌握，首先我们需要在build.gradle中引入Asm，如下：

  ```groovy
  dependencies {
      //核心api，提供visitor api
      implementation 'org.ow2.asm:asm:7.0'
      //可选，提供了一些基于核心api的预定义类转换器
      implementation 'org.ow2.asm:asm-commons:7.0'
      //可选，提供了一些基于核心api的工具类
      implementation 'org.ow2.asm:asm-util:7.0'
  }
  ```

### 读取、访问一个类

读取类之前，首先介绍一下ClassVisitor中的visitXX方法，ClassVisitor的主要结构如下：

```java
public abstract class ClassVisitor {

    //ASM的版本, 版本数值定义在Opcodes接口中，最低为ASM4，最新为ASM7
    protected final int api;
  
    //委托的ClassVisitor，可传空
    protected ClassVisitor cv;

    public ClassVisitor(final int api) {
        this(api, null);
    }
  
    public ClassVisitor(final int api, final ClassVisitor cv) {
        //...
        this.api = api;
        this.cv = cv;
    }

    //表示开始访问这个类
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        if (cv != null) {
            cv.visit(version, access, name, signature, superName, interfaces);
        }
    }

    //表示访问这个类的源文件名(如果有的话)
    public void visitSource(String source, String debug) {
        if (cv != null) {
            cv.visitSource(source, debug);
        }
    }

    //表示访问这个类的外部类(如果有的话)
    public void visitOuterClass(String owner, String name, String desc) {
        if (cv != null) {
            cv.visitOuterClass(owner, name, desc);
        }
    }

    //表示访问这个类的注解(如果有的话)
    public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
        if (cv != null) {
            return cv.visitAnnotation(desc, visible);
        }
        return null;
    }

    //表示访问这个类的内部类(如果有的话)
    public void visitInnerClass(String name, String outerName,
            String innerName, int access) {
        if (cv != null) {
            cv.visitInnerClass(name, outerName, innerName, access);
        }
    }

    //表示访问这个类的字段(如果有的话)
    public FieldVisitor visitField(int access, String name, String desc,
            String signature, Object value) {
        if (cv != null) {
            return cv.visitField(access, name, desc, signature, value);
        }
        return null;
    }

    //表示访问这个类的方法(如果有的话)
    public MethodVisitor visitMethod(int access, String name, String desc,
            String signature, String[] exceptions) {
        if (cv != null) {
            return cv.visitMethod(access, name, desc, signature, exceptions);
        }
        return null;
    }

    //表示结束对这个类的访问
    public void visitEnd() {
        if (cv != null) {
            cv.visitEnd();
        }
    }
  
  	//...省略了一些其他visitXX方法
}
```

可以看到，ClassVisitor的所有visitXX方法都把逻辑委托给另外一个ClassVisitor的visitorXX方法，我们知道，当一个类被加载进JVM中时，它的class的大概结构如下：

{% asset_img plugin1.png plugin %}

所以把class文件结构和ClassVisitor中的方法做对比，可以发现，ClassVisitor中除了visitEnd方法，其他visitXX方法的访问都对应class文件的某个结构，如字段、方法、属性等，每个visitXX方法的参数都表示字段、方法、属性等的相关信息，例如：access表示修饰符、signature表示泛型、desc表示描述符、name表示名字或全权限定名，我们还注意到有些visitXX方法会返回一个XXVisitor类实例，这些XXVisitor类里面又会有类似的visitXX方法，这表示外部可以继续调用返回的XXVisitor实例的visitXX方法，从而继续访问相应结构中的子结构，这个后面再解释。

知道了ClassVisitor中方法的作用后，我们自定义一个类，使用**ClassReader**和**ClassVisitor**把这个类的信息读取、打印出来，首先自定义一个名为OuterClass的类，如下：

```java
@Deprecated
public class OuterClass{

    private int mData = 1;

    public OuterClass(int data){
        this.mData = data;
    }

    public int getData(){
        return mData;
    }

    class InnerClass{ }
}
```

OuterClass类有注解、字段、方法、内部类，然后再自定义一个名为PrintClassVisitor的类扩展自ClassVisitor，如下：

```java
public class PrintClassVisitor extends ClassVisitor implements Opcodes {

    public ClassPrinter() {
        super(ASM7);
    }

    @Override
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
        System.out.println(name + " extends " + superName + "{");
    }

    @Override
    public void visitSource(String source, String debug) {
        System.out.println(" source name = " + source);
    }

    @Override
    public void visitOuterClass(String owner, String name, String descriptor) {
        System.out.println(" outer class = " + name);
    }

    @Override
    public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
        System.out.println(" annotation = " + descriptor);
        return null;
    }

    @Override
    public void visitInnerClass(String name, String outerName, String innerName, int access) {
        System.out.println(" inner class = " + name);
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        System.out.println(" field = "  + name);
        return null;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        System.out.println(" method = " + name);
        return null;
    }

    @Override
    public void visitEnd() {
        System.out.println("}");
    }
}
```

其中Opcodes中定义了很多常量，ASM7就是来自Opcodes，在每个visitXX方法把类的相关信息打印出来，最后使用ClassReader读取OuterClass的class字节码，在accept方法中传入ClassVisitor实例，完成对OuterClass的访问，如下：

```java
public static void main(String[] args) throws IOException {
  //创建ClassVisitor实例
  ClassPrinter printClassVisitor = new ClassPrinter();
  //从构造传入OuterClass的全权限定名，ClassReader会读取OuterClass字节码为字节数组
  ClassReader classReader = new ClassReader(OuterClass.class.getName());
  //在ClassReader的accept传入ClassVisitor实例，开启访问，第二个参数表示访问模式，先不用管，传入0
  classReader.accept(printClassVisitor, 0);
}

运行输出：
com/example/plugindemo/OuterClass extends java/lang/Object{
 source name = OuterClass.java
 annotation = Ljava/lang/Deprecated;
 inner class = com/example/plugindemo/OuterClass$InnerClass
 field = mData
 method = <init>
 method = getData
}
```

ClassReader的构造除了可以接受类的全限定名，还可以接受class文件的输入流，最终都是把class字节码读取到内存中，变成字节数组，ClassReader的accept方法会利用内存偏移量解析构造中读取到的class字节码的字节数组，把class字节码的结构信息从字节数组中解析出来，然后调用传入的ClassVisitor实例的visitorXX方法来访问解析出来的结构信息，而且从运行输出的结果可以看出，accept方法中对于ClassVisitor的visitorXX方法的调用会有一定的顺序，以visit方法开头，以visitEnd方法结束，中间穿插调用其他的visitXX方法，其大概顺序如下：

```
visit 
[visitSource] 
[visitOuterClass] 
[visitAnnotation]
[visitInnerClass | visitField | visitMethod]
visitEnd
```

其中[]表示可选，｜表示平级。

### 生成一个类

前面知道了ClassReader可以用来读取一个类，ClassVisitor可以用来访问一个类，而ClassWirter它可以凭空生成一个类，接下来我们来生成一个名为Person的接口，该接口结构如下：

```java
public interface Person {
    String NAME = "rain9155";
    int getAge();
}
```

使用ClassWriter生成Person接口的代码如下：

```java
import static org.objectweb.asm.Opcodes.*;

public class Main {

  public static void main(String[] args){
    //创建一个ClassWriter，构造传入修改类的行为模式，传0就行
    ClassWriter classWriter = new ClassWriter(0);
    //生成类的头部
    classWriter.visit(V1_7, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE, "com/example/plugindemo/Person", null, "java/lang/Object", null);
    //生成文件名
    classWriter.visitSource("Person.java", null);
    //生成名为NAME，值为rain9155的字段
    FieldVisitor fileVisitor = classWriter.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "NAME", "Ljava/lang/String;", null, "rain9155");
    fileVisitor.visitEnd();
		//生成名为getAge，返回值为int的方法
    MethodVisitor methodVisitor = classWriter.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "getAge", "()I", null, null);
    methodVisitor.visitEnd();
		//生成类完毕
    classWriter.visitEnd();
    //生成的类可以通过toByteArray方法以字节数组形式返回
    byte[] bytes = classWriter.toByteArray();
  }
```

ClassWirter继承自ClassVisitor，它扩展了ClassVisitor的visitorXX方法，使得它具有**生成class字节码**的能力，最终toByteArray方法返回的字节数组可以通过ClassLoader动态加载为一个Class对象，由于我这里生成的是一个接口，所以getAge方法没有方法体，所以visitMethod方法返回的MethodVisitor只是简单的调用了visitEnd就完成了getAge方法头的生成，如果需要生成getAge方法的内部逻辑，例如：

```java
int getAge(){
  return 1;
}
```

那么在调用MethodVisitor的visitEnd方法之前，还需要调用MethodVisitor的其他visitXX方法来生成方法的内部逻辑，MethodVisitor的visitXX方法就是在模拟的JVM的字节码指令，例如入栈、出栈等，对于visitField方法返回的FieldVisitor和visitAnnotation方法返回的AnnotationVisitor的含义和MethodVisitor类似。

可以看到使用ClassWirter生成一个简单的接口的代码量就如此繁琐，如果这是一个类，并且类中的方法有方法体，代码会更加的复杂，所幸的是我们可以通过[ASM Bytecode Outline](https://plugins.jetbrains.com/auth#access_token=1597742367104.6ee795af-5472-40b9-b586-e882ffbff43d.7334cf3f-10b4-47a4-9e77-07e382775999.0-0-0-0-0%3B1.MC0CFQCMwSIpa4MbttWs1X4KLFJ%2BhHK7IAIUDDKsMI4XaK2gN2DfsT21DYjWgGc%3D&token_type=Bearer&expires_in=3600&scope=0-0-0-0-0&state=f0b84383-ffef-42c2-81a4-d201d8cdf52f)插件来完成这繁琐的过程，首先你要在你的AS或IntelliJ IDE中安装这个插件，然后在你想要查看的Asm代码的类**右键 -> Show Bytecode outline**，就会在侧边窗口中显示这个类的字节码(Bytecode)和Asm代码(ASMified)，点击ASMified栏目就会显示这个类的Asm码，例如下图就是Person接口的通过插件生成的Asm代码：

{% asset_img plugin2.png plugin %}

可以看到，使用ClassWriter来生成Person接口。

### 转换一个类

ClassReader可以用来读取一个类，ClassVisitor可以用来访问一个类，ClassWirter可以生成一个类，所以当把它们三个组合在一起时，我们可以把**class字节码通过ClassReader读取，把读取到的class字节码通过扩展的ClassVisitor转换，转换后，再通过ClassWirter重新生成这个类**，就可以达到转换一个类的目的，下面我们把前面的OuterClass类的注解通过转换移除掉，首先自定义一个ClassVisitor，如下：

```java
public class RemoveAnnotationClassVisitor extends ClassVisitor implements Opcodes {

    public RemoveAnnotationClassVisitor(ClassVisitor classVisitor) {
        super(ASM7, classVisitor);
    }

    @Override
    public AnnotationVisitor visitAnnotation(String descriptor, boolean visible) {
      //返回null
      return null;
    }
}
```

这里我只重写了ClassVisitor的visitAnnotation方法，在visitAnnotation方法中返回null，这样调用者就无法使用返回的AnnotationVisitor生成类的注解，然后使用这个RemoveAnnotationClassVisitor，如下：

```java
public static void main(String[] args) throws IOException {
  	//读取OuterClass类的字节码到ClassReader
		ClassReader classReader = new ClassReader(OuterClass.class.getName());
  	//定义用于生成类的ClassWriter
  	ClassWriter classWriter = new ClassWriter(0);
    //把ClassWriter传进RemoveAnnotationClassVisitor的构造中
  	RemoveAnnotationClassVisitor removeAnnotationClassVisitor = new RemoveAnnotationClassVisitor(classWriter);
    //在ClassReader的accept方法中传入RemoveAnnotationClassVisitor实例，开启访问
 	  classReader.accept(removeAnnotationClassVisitor, 0);
    //最终使用ClassWriter的toByteArray方法返回转换后的OuterClass类的字节数组
  	byte[] bytes = classWriter.toByteArray();
}
```

上面这段代码只是把前面所讲的读取、访问、生成一个类的知识结合在一起，ClassVisitor的构造可以传进一个ClassVisitor，从而代理传进的ClassVisitor，而ClassWriter是继承自ClassVisitor的，所以RemoveAnnotationClassVisitor代理了ClassWriter，RemoveAnnotationClassVisitor把OuterClass转换完后就交给了ClassWriter，最终我们可以通过ClassWriter的toByteArray方法返回转换后的OuterClass类的字节数组。

上面是只有简单的一个ClassVisitor进行转换的代码，如果我们把它扩展，我们还可以定义RemoveMethodClassVisitor、AddFieldClassVisitor等多个具有不同功能的ClassVisitor，然后把所有的ClassVisitor串成一条**转换链**，把ClassReader想象成头，ClassWriter想象成尾，中间是一系列的ClassVisitor，ClassReader把读取到的class字节码经过一系列的ClassVisitor转换后到达ClassWriter，最终被ClassWriter生成新的class，这个过程如图：

{% asset_img plugin3.png plugin %}

Asm的入门知识就讲解到这里，如果想要了解更多关于Asm的知识请查阅开头给出的官方教程，下面我们来学习Transform相关知识。

## Transform

官网：[Transform](https://developer.android.com/reference/tools/gradle-api/com/android/build/api/transform/Transform)

Transform是android gradle api中的一部分，它可以在android项目的.class文件编译为.dex文件之前，得到所有的.class文件，然后我们可以在Transform中对所有的.class文件进行处理，所以Transform提供了一种可以让我们得到android项目的字节码的能力，如图红色标志的位置为Transform的作用点：

{% asset_img plugin4.png plugin %}

上图就是android打包流程的一部分，而android的打包流程是交给android gradle plugin完成的，所以如果我们想要自定义Transform，必须要注入到android gradle plugin中才能产生效果，而plugin的执行单元是Task，但Transform并不是Task，那么Transform是怎么被执行的呢？android gradle plugin会为每一个Transform创建对应的TransformTask，由相应的TransformTask执行相应的Transform，

接下来我们来介绍Transform，首先我们需要在build.gradle中引入Transform，如下：

```groovy
dependencies {
   	//引用android gradle api, 里面包含transform api
    implementation 'com.android.tools.build:gradle:4.0.0'
}
```

因为transform api是android gradle api的一部分，所以我们引入android gradle api就行，自定义一个名为MyTransform的Transform，如下：

```java
public class MyTransform extends Transform {

    @Override
    public String getName() {
        //用来生成TransformTask的名称
        return "MyTransform";
    }

    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        //输入类型
        return TransformManager.CONTENT_CLASS;
    }

    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        //输入的作用域
        return TransformManager.SCOPE_FULL_PROJECT;
    }
  
   @Override
    public boolean isIncremental() {
        //是否开启增量编译
        return false;
    }
  
    @Override
    public void transform(TransformInvocation transformInvocation){
        //在这里处理class文件
    }
}
```

Transform是一个抽象类，所以它会强制要求我们实现几个方法，还要重写transform方法，下面分别讲解这几个方法的含义：

### getName方法

前面讲过android gradle plugin会为每一个Transform创建一个对应的TransformTask，而创建的TransformTask的名称一般的格式为transformClassesWith**XX1**For**XX2**，其中XX1的值就是getName方法的返回值，而XX2的值就是当前构建环境的Build Variants，例如Debug、Release等，所以如果你自定义的的Transform名为MyTransform，Build Variants为Debug，那么该Transform对应的Task名为transformClassesWithMyTransformForDebug。

### getInputTypes和getScopes方法

getInputTypes方法和getScopes方法都返回一个Set集合，其中集合的元素类型分别为ContentType接口和Scope枚举，在Transform中，**ContentType**表示Transform输入的**类型**，**Scope**表示Transform输入的**作用域**，Transform从ContentType和Scope这两个维度来**过滤**Transform的输入，某个输入只有同时满足了getInputTypes方法返回的ContentType集合和getScopes方法返回的Scope集合，才会被Transform消费。

在Transform中，主要有两种类型的输入，它们分别为CLASSES和RESOURCES，以实现了ContentType接口的枚举[DefaultContentType](https://developer.android.com/reference/tools/gradle-api/com/android/build/api/transform/QualifiedContent.DefaultContentType)表示，各枚举含义如下：

| DefaultContentType | 含义                            |
| ------------------ | ------------------------------- |
| CLASSES            | 表示在jar或文件夹中的.class文件 |
| RESOURCES          | 表示标准的java源文件            |

同理，在Transform中，输入的作用域也以枚举[Scope](https://developer.android.com/reference/tools/gradle-api/com/android/build/api/transform/QualifiedContent.Scope)表示，主要有PROJECT、SUB_PROJECTS、EXTERNAL_LIBRARIES、TESTED_CODE、PROVIDED_ONLY这五种作用域，各枚举含义如下：

| Scope              | 含义                                    |
| ------------------ | --------------------------------------- |
| PROJECT            | 只处理当前项目                          |
| SUB_PROJECTS       | 只处理当前项目的子项目                  |
| EXTERNAL_LIBRARIES | 只处理当前项目的外部依赖库              |
| TESTED_CODE        | 只处理当前项目构建环境的测试代码        |
| PROVIDED_ONLY      | 只处理当前项目使用provided-only依赖的库 |

ContentType和Scope都可以分别进行组合，已Set集合的形式返回，在**TransformManager**类中定义了一些我们常用的组合，我们可以直接使用，如MyTransform的ContentType为**CONTENT_CLASS**， Scope为**SCOPE_FULL_PROJECT**，定义如下：

```java
public class TransformManager extends FilterableStreamCollection {
  
      public static final Set<ContentType> CONTENT_CLASS = ImmutableSet.of(CLASSES);
  		
      public static final Set<ScopeType> SCOPE_FULL_PROJECT = ImmutableSet.of(Scope.PROJECT, Scope.SUB_PROJECTS, Scope.EXTERNAL_LIBRARIES);
  
  	 //...还有其他很多组合
}
```

可以看到CONTENT_CLASS由CLASSES组成，SCOPE_FULL_PROJECT由PROJECT、SUB_PROJECTS、EXTERNAL_LIBRARIES组成，所以MyTransform只会处理来自当前项目(包括子项目)和外部依赖库的.class文件输入。

### isIncremental方法

isIncremental方法的返回值表示当前Transform是否开启**增量编译**，返回true表示开启，增量编译是相对于全量编译而言，全量编译是指当Gradle每一次构建时，Transform都会产生新的输出而不管之前的输出是否存在，而开启增量编译后，当Gradle每一次构建时，Transform如果发现本次输入与上一次输入没有发生变化时，就判断本次编译为增量编译，判定为增量编译后，在transform方法中就可以根据输入的**[Status](https://developer.android.com/reference/tools/gradle-api/com/android/build/api/transform/Status)**来处理每个输入的文件，Status也是一个枚举，各枚举含义如下：

| Status     | 含义                                 |
| ---------- | ------------------------------------ |
| NOTCHANGED | 该文件自上次构建以来没有发生变化     |
| ADDED      | 该文件为新增文件                     |
| CHANGED    | 该文件自上次构建以来发生变化(被修改) |
| REMOVED    | 该文件已被删除                       |

开启增量编译可以大大的提高Gradle的构建速度。

>  注意：如果你开启了增量编译，那么自定义的Transform的transform方法中必须提供对增量编译的支持，即根据Status来对输入的文件作出处理，否则增量编译是不生效的，这在后面的插件实现中可以看到如何提供对增量编译的支持。

### transform方法

transform方法就是Transform中处理输入的地方，TransformTask执行时就是执行Transform的transform方法，transform方法的参数是**TransfromInvocation**，它包含的当前Transform的输入和输出信息，可以使用TransfromInvocation的**getInputs**方法来获取Transform的输入，使用TransformInvocation的**getOutputProvider**方法来生成Transform的输出，还可以通过TransfromInvocation的**isIncremental**方法的返回值判断本次transform是否是增量编译。

TransfromInvocation的getInputs方法返回一个元素类型为**TransformInput**的集合，其中TransformInput可以获取两种类型的输入，如下：

```java
public interface TransformInput {
  	//getJarInputs方法返回JarInput集合
    Collection<JarInput> getJarInputs();
  
  	//getDirectoryInputs方法返回DirectoryInput集合
    Collection<DirectoryInput> getDirectoryInputs();
}
```

两种类型的输入又抽象为**JarInput**和**DirectoryInput**，JarInput代表输入为.Jar文件，DirectoryInput代表输入为文件夹类型，JarInput有一个**getStatus**方法来获取该jar文件的Status，而DirectoryInput**getChangedFiles**方法来获取一个Map<File, Status>集合，所以可以遍历这个Map集合，然后根据File对应的Status来对File进行处理。

TransfromInvocation的getOutputProvider方法返回一个**TransformOutputProvider**，它可以用来创建Transform的输出位置，如下：

```java
public interface TransformOutputProvider {
    //删除所有输出
    void deleteAll() throws IOException;

    //根据参数给的name、ContentType、Scope、Format来创建输出位置
    File getContentLocation(
            @NonNull String name,
            @NonNull Set<QualifiedContent.ContentType> types,
            @NonNull Set<? super QualifiedContent.Scope> scopes,
            @NonNull Format format);
}

```

调用getContentLocation方法就可以创建一个输出位置并返回该位置代表的File实例，如果存在就直接返回，通过getContentLocation方法创建的输出位置一般位于**/app/build/intermediates/transforms/Transform名称/**目录下，其中**Transform名称**就是getName方法的返回值，例如MyTransform的输出位置就是/app/build/intermediates/transforms/MyTransform/目录下，该目录下都是Transform输出的jar文件或文件夹，名称是以0、1、2、...递增的命名形式命名，调用deleteAll方法就可以把getContentLocation方法创建的输出位置下的所有文件删除掉。

所以如果不考虑增量编译的话，transform方法里面一般会这样写：

```java
 public void transform(TransformInvocation transformInvocation) throws IOException {
        //通过TransformInvocation的getInputs方法获取所有输入，是一个集合，TransformInput代表一个输入
        Collection<TransformInput> transformInputs = transformInvocation.getInputs();

        //通过TransformInvocation的getOutputProvider方法获取输出的提供者，通过TransformOutputProvider可以创建Transform的输出
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();

        //遍历所有的输入，每一个输入里面包含jar和directory两种输入类型的文件集合
        for(TransformInput transformInput : transformInputs){
            Collection<JarInput> jarInputs = transformInput.getJarInputs();
            //遍历，处理jar文件
            for(JarInput jarInput : jarInputs){
                File dest = outputProvider.getContentLocation(
                        jarInput.getName(),
                        jarInput.getContentTypes(),
                        jarInput.getScopes(),
                        Format.JAR
                );
                //这里只是简单的把jar文件复制到输出位置
                FileUtils.copyFile(jarInput.getFile(), dest);
            }

            Collection<DirectoryInput> directoryInputs = transformInput.getDirectoryInputs();
            //遍历，处理文件夹
            for(DirectoryInput directoryInput : directoryInputs){
                File dest = outputProvider.getContentLocation(
                        directoryInput.getName(),
                        directoryInput.getContentTypes(),
                        directoryInput.getScopes(),
                        Format.DIRECTORY
                );
                //这里只是简单的把文件夹中的所有文件递归地复制到输出位置
                FileUtils.copyDirectory(directoryInput.getFile(), dest);
            }
        }
    }
```

就是获取到输入，遍历输入中的所有JarInput和DirectoryInput，然后把相应的输入简单地重定向到输出位置中，在这过程中，我们还可以获取jar文件和文件夹中的class文件，对class文件进行修改后再进行重定向到输出，这就达到了在编译期间修改字节码的目的，这也是后面插件实现的核心，该Transform的输出会作为下一个Transform的输入，如下：

{% asset_img plugin5.png plugin %}

现在对于Asm和Transform都有了一个大概的了解，就可以动手实现函数耗时检测插件。

## 插件实现

检测函数耗时很简单，只需要在每个方法的开头和结尾增加耗时检测的代码逻辑即可，例如：

```java
protected void onCreate(Bundle savedInstanceState) {
  long startTime = System.currentTimeMillis();//start
  super.onCreate(savedInstanceState);
  setContentView(R.layout.activity_main);
  long endTime = System.currentTimeMillis();//end
  long costTime = endTime - startTime;
  if(costTime > 100){
    StackTraceElement thisMethodStack = (new Exception()).getStackTrace()[0];//获得当前方法的StackTraceElement
    Log.e("TimeCost", String.format(
      "===> %s.%s(%s:%s)方法耗时 %d ms",
      thisMethodStack.getClassName(), //类的全限定名称
      thisMethodStack.getMethodName(),//方法名
      thisMethodStack.getFileName(),  //类文件名称
      thisMethodStack.getLineNumber(),//行号
      costTime                        //方法耗时
    	)
    );
  }
}
```

上面在onCreate方法的开头和结尾记录onCreate方法的运行时间，当运行时间大于100ms时就打印出来，打印效果如下：

{% asset_img plugin6.png plugin %}

打印的log会显示方法的详细信息，点击行号可以定位到耗时函数处。

我们不可能手动的替应用内的每个方法都加上上述代码，所以我们需要Gradle插件替我们完成这重复的过程，在项目编译的过程中，通过Transform拿到项目中每个类的字节码，然后使用Asm对每个类的的每个方法增加上述函数耗时检测的字节码，如果你不知道自定义一个Gradle插件的步骤，请移步上一篇文章，我把Gradle插件的实现代码放在**buildSrc**目录下，整个项目的目录结构如下：

{% asset_img plugin7.png plugin %}

有关Plugin和Transform实现的代码放在com.example.plugin下，有关Asm实现的代码放在com.example.asm下。

###自定义Plugin和Transform







###处理class文件













































## 结语

参考资料：

[java class结构](https://www.jianshu.com/p/ae3f860499aa)

[Android Gradle Plugin打包Apk过程中的Transform API](https://www.jianshu.com/p/811b0d0975ef)

[一起玩转Android项目中的字节码](http://quinnchen.cn/2018/09/13/2018-09-13-asm-transform/#timing-plugin)

[一文读懂 AOP](https://juejin.im/post/6844903728525361165#heading-2)

[Gradle 篇 -- Android Gradle Plugin 主要流程分析](https://juejin.im/post/6844903844749508622#heading-9)

