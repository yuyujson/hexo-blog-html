---
title: io流
permalink: io-stream
date: 2018-03-06
tags:
categories: io
---
# 分类一：按操作方式（类结构）
**字节流和字符流：**
- 字节流：以字节为单位，每次次读入或读出是8位数据。可以读任何类型数据。
- 字符流：以字符为单位，每次次读入或读出是16位数据。其只能读取字符类型数据。

<!--more-->

**输出流和输入流：**
- 输出流：从内存读出到文件。只能进行写操作。
- 输入流：从文件读入到内存。只能进行读操作。

**注意：** 这里的出和入，都是相对于系统内存而言的。

**节点流和处理流：**
- 节点流：直接与数据源相连，读入或读出。
- 处理流：与节点流一块使用，在节点流的基础上，再套接一层，套接在节点流上的就是处理流。

**为什么要有处理流？**

直接使用节点流，读写不方便，为了更快的读写文件，才有了处理流。



## 分类说明：
**1. 输入字节流InputStream：**
- **ByteArrayInputStream、StringBufferInputStream、FileInputStream** 是三种基本的介质流，它们分别从Byte 数组、StringBuffer、和本地文件中读取数据。

- **PipedInputStream** 是从与其它线程共用的管道中读取数据。PipedInputStream的一个实例要和PipedOutputStream的一个实例共同使用，共同完成管道的读取写入操作。主要用于线程操作。

- **DataInputStream：** 将基础数据类型读取出来

- **ObjectInputStream** 和所有 FilterInputStream 的子类都是装饰流（装饰器模式的主角）。

**2. 输出字节流OutputStream：**
- **ByteArrayOutputStream、FileOutputStream：** 是两种基本的介质流，它们分别向- Byte 数组、和本地文件中写入数据。

- **PipedOutputStream** 是向与其它线程共用的管道中写入数据。

- **DataOutputStream** 将基础数据类型写入到文件中

- **ObjectOutputStream** 和所有 FilterOutputStream 的子类都是装饰流。



**3. 字符输入流Reader:**
- **FileReader、CharReader、StringReader** 是三种基本的介质流，它们分在本地文件、Char 数组、String中读取数据。

- **PipedReader：** 是从与其它线程共用的管道中读取数据

- **BufferedReader ：** 加缓冲功能，避免频繁读写硬盘

- **InputStreamReader：** 是一个连接字节流和字符流的桥梁，它将字节流转变为字符流。


**4. 字符输出流Writer：**

- **StringWriter:** 向String 中写入数据。

- **CharArrayWriter：** 实现一个可用作字符输入流的字符缓冲区

- **PipedWriter:** 是向与其它线程共用的管道中写入数据

- **BufferedWriter ：** 增加缓冲功能，避免频繁读写硬盘。
- 
- **PrintWriter 和PrintStream** 将对象的格式表示打印到文本输出流。 极其类似，功能和使用也非常相似

- **OutputStreamWriter：** 是OutputStream 到Writer 转换的桥梁，它的子类FileWriter 其实就是一个实现此功能的具体类（具体可以研究一SourceCode）。功能和使用和OutputStream 极其类似，后面会有它们的对应图。


## 分类说明：

**对文件进行操作（节点流）：**

- FileInputStream（字节输入流），

- FileOutputStream（字节输出流），

- FileReader（字符输入流），

- FileWriter（字符输出流）

**对管道进行操作（节点流）：**

- PipedInputStream（字节输入流）,

- PipedOutStream（字节输出流），

- PipedReader（字符输入流），

- PipedWriter（字符输出流）。

- PipedInputStream的一个实例要和PipedOutputStream的一个实例共同使用，共同完成管道的读取写入操作。主要用于线程操作。

**字节/字符数组流（节点流）：**

- ByteArrayInputStream，

- ByteArrayOutputStream，

- CharArrayReader，

- CharArrayWriter；

**除了上述三种是节点流，其他都是处理流，需要跟节点流配合使用。**

**Buffered缓冲流（处理流）**：带缓冲区的处理流，缓冲区的作用的主要目的是：避免每次和硬盘打交道，提高数据访问的效率。

- BufferedInputStream，

- BufferedOutputStream，

- BufferedReader,

- BufferedWriter,

**转化流（处理流）：**

- InputStreamReader：把字节转化成字符；

- OutputStreamWriter：把字节转化成字符。

**基本类型数据流（处理流）：用于操作基本数据类型值。**

因为平时若是我们输出一个8个字节的long类型或4个字节的float类型，那怎么办呢？可以一个字节一个字节输出，也可以把转换成字符串输出，但是这样转换费时间，若是直接输出该多好啊，因此这个数据流就解决了我们输出数据类型的困难。数据流可以直接输出float类型或long类型，提高了数据读写的效率。

- DataInputStream，

- DataOutputStream。

**打印流（处理流）：**

一般是打印到控制台，可以进行控制打印的地方。

- PrintStream，

- PrintWriter，

**对象流（处理流）：**

把封装的对象直接输出，而不是一个个在转换成字符串再输出。

- ObjectInputStream，对象反序列化；

- ObjectOutputStream，对象序列化；

**合并流（处理流）：**

- SequenceInputStream：可以认为是一个工具类，将两个或者多个输入流当成一个输入流依次读取。

**其他类：File（已经被Java7的Path取代）**

File类是对文件系统中文件以及文件夹进行封装的对象，可以通过对象的思想来操作文件和文件夹。 File类保存文件或目录的各种元数据信息，包括文件名、文件长度、最后修改时间、是否可读、获取当前文件的路径名，判断指定文件是否存在、获得当前目录中的文件列表，创建、删除文件和目录等方法。

**其他类：RandomAccessFile**

该对象并不是流体系中的一员，其封装了字节流，同时还封装了一个缓冲区（字符数组），通过内部的指针来操作字符数组中的数据。 该对象特点：

- 该对象只能操作文件，所以构造函数接收两种类型的参数：a.字符串文件路径；b.File对象。

- 该对象既可以对文件进行读操作，也能进行写操作，在进行对象实例化时可指定操作模式(r,rw)。

**注意：**

IO中的很多内容都可以使用NIO完成，这些知识点大家知道就好，使用的话还是尽量使用NIO/AIO。