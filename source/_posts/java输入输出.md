---
title: java学习总结之I/O操作
tags: I/O
categories: java
date: 2020-05-30 23:20:42
---


## 前言

I/O(Input/Output)操作，即输入输出操作，它是一个相对的过程，我们一般站在位于内存中的程序的角度来思考这个操作的过程，**输入**就是程序需要数据，把数据从数据源中流入程序，**输出**就是程序需要保存或传输数据，把数据从程序中流出到数据源，这个**数据源**一般为文件、网络、压缩包等，所以数据源和程序就通过数据流通道组成了一个半双工的输入/输出模式，如下：

{% asset_img io1.png I/O1 %}

java从更高层次把输入和输出抽象出来，封装成一些对程序员友好的API，我们只需要操作这些I/O接口就能简单的实现I/O操作，了解了输入输出模式的大体结构后，下面进入正文讲解。

## I/O框架

{% asset_img io2.png I/O1 %}

有关I/O的类主要都在java .io包中，从上图可以看出，java中的I/O流根据数据的单位划分可以分为字节流和字符流，两类都有输入和输出操作：

- **字节流**：字节流主要用来处理字节(byte)或二进制对象，字节流的标志就是以Stream结尾，在字节流中，输入数据使用InputStream，输出数据使用OutputStream，InputStream/OutputStream是其他所有输入/输出字节流的抽象父类；
- **字符流**：字符流主要用来处理字符(char)或字符串，字符流的标志就是以Reader或Writer结尾，在字符流中，输入数据使用Reader，输出数据使用Writer，Reader/Writer是其他所有输入/输出字符流的抽象父类.

不管是字节流还是字符流，它们都有一个close方法，用来关闭资源，因为它们都实现了Closeable接口，所以我们在通过这些流对象操作了文件后，记得调用close方法关闭资源，不然流对象就会一直关联着文件，导致文件无法释放，造成内存泄漏。

> 在磁盘上保存或在网络上传输的数据最终都是二进制字节数据而不是字符数据，之所以存在字符流，是因为字符流中增加了对编码的处理，对多国语言支持比较好，所以如果需要处理文本、字符串等建议使用字符流，但是如果是处理音频、视频、图片等建议使用字节流。

## 字节流

### 1、InputStream

{% asset_img io3.png I/O1 %}

InputStream是所有字节输入流的抽象父类，为所有字节输入流提供一个标准的、基本的读取字节的方法，它常用方法如下：

| 方法                                 | 描述                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| int read()                           | 从输入流中读取下一个字节数据(0~255)，已到达最后没有可读取数据时返回-1 |
| int read(byte[] b)                   | 从输入流中读取b.length个字节到数组b中，并返回实际读取的字节数，已到达最后没有可读取数据时返回-1 |
| int read(byte[] b, int off, int len) | 读取字节保存在b数组的下标off到len - 1处，已到达最后没有可读取数据时返回-1 |
| int available()                      | 返回可以从输入流中读取的字节数(估计值)                       |

InputStream的常用子类如下：

| InputStream子类     | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| FileInputStream     | 文件字节输入流，用于从文件中读取信息                         |
| ByteArrayInputSream | 字节数组输入流，允许将内存的缓冲区作为输入流使用，用于读取其内置缓存字节数组中的字节 |
| ObjectInputStream   | 对象字节输入流，从输入流中读取对象                           |
| BufferedInputStream | 缓冲字节输入流，使用缓冲区的输入流，它常常用来装饰其他字节输入流，为其提供缓冲功能，提高访问效率 |
| DataInputStream     | 数据字节输入流，可以从输入流中读取基本的数据类型，如int、char、long等 |

使用InputStream读取数据的大概流程如下：

```java
void main(String[] args) throws IOException{
    InputStream in = new FileInputStream("file.txt");
    int size = in.available();
    byte[] b = new byte[size];
    int len = in.read(b, 0, size);
    String value = new String(b);
    System.out.println("读取出的内容：value = " + value);
    in.close();
}
```

### 2、OutputStream

{% asset_img io4.png I/O1 %}

OutputStream是所有字节输出流的抽象父类，为所有字节输出流提供一个标准的、基本的写入字节的方法，它常用方法如下：

| 方法                                   | 描述                                             |
| -------------------------------------- | ------------------------------------------------ |
| void write()                           | 将指定的字节写入到输出流中                       |
| void write(byte[] b)                   | 将字节数组b的内容写入到输出流中                  |
| void write(byte[] b, int off, int len) | 将字节数组中下标off到len - 1的字节写入到输出流中 |
| void flush()                           | 清除输出流中的缓存，强制写出任何输出缓冲中的字节 |

OutputStream的常用子类如下：

| OutputStream子类     | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| FileOutputStream     | 文件字节输出流，用于把字节写入到文件中                       |
| ByteArrayOutputSream | 字节数组输出流，允许将内存的缓冲区作为输出流使用，用于把字节写入到内置缓存字节数组中 |
| ObjectOutputStream   | 对象字节输出流，用于把对象写入到输出流中                     |
| BufferedOutputStream | 缓冲字节输出流，使用缓冲区的输出流，它常常用来装饰其他字节输出流，为其提供缓冲功能，提高访问效率 |
| DataOutputStream     | 数据字节输出流，可以把基本的数据类型写入到输出流中           |

使用OutputStream写入数据的大概流程如下：

```java
void main(String[] args)  throws IOException{
    OutputStream os = new FileOutputStream("file.txt", true);
    String content = "待写入的内容";
    os.write(content.getBytes());
    os.flush();
    os.close();
}
```

## 字符流

### 1、Reader

{% asset_img io5.png I/O1 %}

 Reader是所有字符输入流的抽象父类，为所有字符输入流提供一个标准的、基本的读取字符的方法，它常用方法如下：

| 方法                                     | 描述                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| int read()                               | 从输入流中读取下一个字符(0~65535)，已到达最后没有可读取数据时返回-1 |
| int read(char[] cbuf)                    | 从输入流中读取b.length个字符到数组b中，并返回实际读取的字符数，已到达最后没有可读取数据时返回-1 |
| int read(char[] cbuffer,int off,int len) | 读取字符保存在b数组的下标off到len - 1处，已到达最后没有可读取数据时返回-1 |
| boolean ready()                          | 判断输入流是否做好读的准备                                   |

Reader的常用子类如下：

| Reader子类        | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| CharArrayReader   | 字符数组输入流，实现了字符缓冲区的输入流，将其内置字符缓存数组中的数据（通过构造方法传入）读取到程序中 |
| InputStreamReader | 字节转换流，用于将字节流转换成字符流，它读取字节，并使用指定的charset将其解码为字符 |
| FileReader        | 文件字符输入流，以字符的形式读取文件中的内容                 |
| BufferedReader    | 缓冲字符输入流，它是一个装饰类，装饰其他字符输入流，为其提供缓冲功能，提高效率 |

使用Reader读取数据的大概流程如下：

```java
void main(String[] args)  throws IOException{
    FileReader reader = new FileReader("file.txt");
    BufferedReader bReader = new BufferedReader(reader);
    StringBuilder content = new StringBuilder();
    String line;
    //readLine方法是BufferedReader中的方法，它一次读取一行，比read方法一次读取一个字符的效率快很多
    while((line = bReader.readLine()) != null){
        content.append(line);
        content.append("\r\n");
    }
    System.out.println("读取到的内容：content" + content);
    bReader.close();
    reader.close();
}

   //将字节流转换为Buffer字符流，同样可以使用readLine方法一行一行地读
public void main(String[] args)  throws IOException{
    StringBuilder content = new StringBuilder();
    String line;
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(new FileInputStream("file.txt"))
    );
    while((line = reader.readLine()) != null){
        content.append(line);
        content.append("\r\n");
    }
    System.out.println("读取到的内容：content" + content);
    reader.close();
}
```

### 2、Writer

{% asset_img io6.png I/O1 %}

Writer是所有字符输出流的抽象父类，为所有字符输出流提供一个标准的、基本的写入字符的方法，它常用方法如下：

| 方法                                       | 描述                                             |
| ------------------------------------------ | ------------------------------------------------ |
| void write()                               | 将指定字符写入到输出流中                         |
| void write(char[] cbuf)                    | 将字符数组b的内容写入到输出流中                  |
| void write(char[] cbuffer,int off,int len) | 将字符数组中下标off到len - 1的字符写入到输出流中 |
| void flush()                               | 将缓存区中的所有数据写到文件中                   |

Writer的常用子类如下：

| Reader子类        | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| CharArrayWriter   | 字符数组输出流，实现了字符缓冲区的输出流，将字符写入到内置的字符缓存数组中 |
| InputStreamWriter | 字节转换流，用于将字节流转换成字符流，它可以使用指定的charset来写入字符 |
| FileWriter        | 文件字符输出流，将字符写入到指定的文件中                     |
| BufferedWriter    | 缓冲字符输出流，它是一个装饰类，装饰其他字符输出流，为其提供缓冲功能，提高效率 |

使用Writer写入数据的大概流程如下：

```java
void main(String[] args){
    FileWriter writer = new FileWriter("file.txt", true);
    BufferedWriter bWriter = new BufferedWriter(writer);
    StringBuilder content = new StringBuilder("待写入的内容");
    writer.write(content.toString());
    bWriter.flush();
    bWriter.close();
    writer.close();
}
```

## RandomAccessFile

RandomAccessFile它是属于IO系统的一部分，在java中，RandomAccessFile用来表示**可随机访问**的文件，前面我们所讲的字符流或字节流都是一种顺序流，通过顺序流打开的文件只能按顺序地从上往下地访问，不能说从后往前，或者说从中间开始访问，而通过RandomAccessFile打开的文件可以让我们不按顺序地读取文件，RandomAccessFile内部其实是通过称为file point的指针来进行文件地读写，我们可以设置指针在文件中的位置，从而达到从任意位置开始读写的目的，它的常用方法如下：

| 方法                                  | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| RandomAccessFile(File f, String mode) | 使用指定的File对象和模式(r/rw)创建随机文件流                 |
| long getFilePointer()                 | 返回以字节计算的文件偏移量(从文件开始计算)，作为下一个read或writer的起点 |
| long length() / setLength(long)       | 返回文件中的字节数 / 为文件设置新的长度                      |
| int read() / int read(byte[])         | 从文件中读取字节数据                                         |
| void seek(long pos)                   | 设置文件偏移量，以字节为单位                                 |
| int skipBytes(int n)                  | 跳过n个字节                                                  |
| void write(byte[] b)                  | 将指定的字节数组写入到文件中，从当前的文件指针开始写         |

下面是它的一些基本使用：

```java
//从指定的位置开始读取数据
public static void main(String[] args) throws IOException {
    File file=new File("file.txt");
    RandomAccessFile randomFile = new RandomAccessFile(file,"r");
    // 获取 RandomAccessFile对象文件指针的位置，初始位置为0
    System.out.print("输入内容：" + randomFile.getFilePointer());
    //移动文件记录指针的位置
    randomFile.seek(1000);
    byte[] b=new byte[1024];
    int hasRead = 0;
    while((hasRead = randomFile.read(b))>0){
        //输出文件读取的内容
        System.out.print(new String(b, 0, hasRead));
    }
    randomFile.close();
}
```

通过RandomAccessFile很容易地实现断点下载。

## File

文件是输入/输出流的操作对象之一，java中用File代表文件，File类中封装了很多对文件或文件夹操作的方法，我把File类中常用的方法分为四类：状态检测、路径和名称操作、文件操作、目录操作。

File中的状态检测方法：

| 方法                 | 描述                                     |
| -------------------- | ---------------------------------------- |
| boolean canExecute() | 判断此File实例对应的文件是否是可执行文件 |
| int length()         | 返回此File实例对应文件的长度，单位为字节 |
| boolean canRead()    | 判断此File实例对应的文件是否可读         |
| boolean canWrite()   | 判断此File实例对应的文件是否可写         |
| boolean isHidden()   | 判断此File实例对应的文件是否是隐藏文件   |
| long lastModified()  | 返回此File实例的最后修改时间             |

File路径和名称操作方法：

| 方法                     | 描述                                     |
| ------------------------ | ---------------------------------------- |
| String getAbsolutePath() | 获取此File实例对应文件或文件夹的绝对路径 |
| String getPath()         | 获取此File实例对应文件或文件夹的相对路径 |
| String getName()         | 获取此File实例对应文件或文件夹的名称     |
| File getParentFile()     | 获取此File实例对应文件的上级目录文件     |
| String getParent()       | 获取此File实例对应文件的上级目录路径     |

File中的文件操作方法：

| 方法                                                         | 描述                                       |
| ------------------------------------------------------------ | ------------------------------------------ |
| boolean exists()                                             | 判断此File实例对应的文件是否存在           |
| boolean isFile()                                             | 判断此File实例是否是一个标准文件           |
| boolean delete()                                             | 删除此File实例对应的文件                   |
| boolean createNewFile()                                      | 根据File实例创建新的文件                   |
| void deleteOnExit()                                          | 在JVM终止时删除File实例对应的文件          |
| boolean renameTo(File dest)                                  | 根据dest重命名此File实例对应的文件         |
| static File createTempFile(String prefix, String suffix, File directory) | 在directory目录下创建prefix.suffix临时文件 |

File中的目录(文件夹)操作方法：

| 方法                                | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| boolean isDirectory()               | 判断此File实例是否是一个文件夹                               |
| boolean mkdir()                     | 根据File实例对应的文件夹创建当前目录（一级目录）             |
| boolean mkdirs()                    | 根据File实例对应的文件夹创建当前目录和上级目录（多级目录）   |
| String[] list()                     | 根据File实例对应的文件夹列出当前目录中所有文件和文件夹的路径 |
| File[] listFiles()                  | 根据File实例对应的文件夹列出当前目录中所有文件和文件夹的File实例 |
| static File[] listRoots()           | 列出可用的文件系统盘符(根路径)                               |
| File[] listFiles(FileFilter filter) | 返回一个File数组，表示当前File实例下满足指定过滤器filter的所有文件和目录 |

我们可以发现File中的方法只能获取或设置文件相应的**信息**，并不能获取或设置文件里面的**内容**，想要获取或设置文件里面的内容需要通过前面讲过的**流操作**来完成，除此之外File还有一些静态字段用来表示与系统无关的属性，例如目录分隔符separator、属性分隔符pathSeparator，这些分隔符在不同的操作系统上的表示是不一样的，例如在window上通过File的separator和pathSeparator获取到的输出是 \ 和 ; ，在Unix上获取到的输出是 / 和 ：。

## 结语

本文从字节流、字符流，文件操作等介绍了java的IO系统，同时这些基于流的IO操作又被称为传统的IO操作，为什么说它是传统的呢，因为早在java4就推出**NIO**，即New IO，NIO相对与传统的IO的主要区别是NIO是基于通道和缓冲区的，同时NIO是非阻塞的，所以它比传统的IO更高效，所以在java4之后，NIO完全可以替代传统的IO，但是话虽是这么说，传统的IO也十分重要，毕竟在一些简单的场景下它的使用还是更加的方便，NIO更多地是使用在高并发的服务端上。

参考资料：

[Java：带你全面了解神秘的Java NIO](https://www.jianshu.com/p/d30893c4d6bb)

[Java IO体系架构图](https://blog.csdn.net/jssg_tzw/article/details/77991749)



