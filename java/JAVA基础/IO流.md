![img](https://pics2.baidu.com/feed/6159252dd42a28341e23dde5587a81ec17cebfa8.jpeg?token=201587ab58af94624ef1a60c83d1e396&s=C8611F7091BFE5CC1C5D95CB000030B2)

![这里写图片描述](https://img-blog.csdn.net/20180127210410630)

**1字符 = 2字节； 1字节(byte) = 8位(bit)； 一个汉字占两个字节长度。一个英文字符占一个字节**

**输入流就是从外部文件输入到内存，输出流主要是从内存输出到文件。**



## 常见面试题

![Java IO运行机制图解](https://www.pianshen.com/images/646/0929aead96ba68f8dce563cf8e6f3a46.JPEG)



1. **什么是IO流， 有几种**

   是指一种数据的流，从源头流向目的地，比如文件拷贝。

   - 方向上：分为输入流和输出流

     输入流从文件中读取数据到内存（process）

     输出流从内存（进程）读取数据然后写到目标文件

   - 种类上：分为字节流和字符流

     字节流继承 InputStream 和 OutputStream

     字符流继承 InputStreamReader 和 OutputStreamReader

2. **字节流和字符流的区别**

   字节流用于操作包含ASCII字符的文件。JAVA也支持其他的字符如Unicode，为了读取包含Unicode字符的文件，JAVA语言引入了字符流。ASCII作为Unicode的子集，对于英语字符的文件，可以使用字节流也可以使用字符流。

3. **Java流中的超类有哪些**

- java.io.InputStream（字节输入流）
- java.io.OutputStream（字节输出流）
- java.io.Reader（字符输入流）
- java.io.Writer（字符输出流）

4. **System.out.println（）是什么**

   println是PrintStream的一个方法，out是一个静态PrintStream类型的成员变量，System是java.lang包中的类，用于和底层的操作系统进行交互

5. **什么是Filter流**

   Filter Stream是一种IO流， 起到对存在的流添加一些额外功能的作用， 如给目标文件增加源文件中不存在的行数，或者增加拷贝的性能。

   有四个常用的filter stream，

   字节：FilterInputStream，FilterOutStream

   字符：FilterReader，FilterWriter



## 字节输入流InputStream（抽象类）

java.io 包下所有的字节输入流都继承自 InputStream，并且实现了其中的方法。InputStream中提供的主要操作方法如下



- int read(): 从输入流中读取一个字节的二进制数据

- int read(byte[] b): 将多个字节读入到数组中，填满整个数组

- int read(byte[] b, int offset, int length): 从输入流中读取长度为len的数据，从数组b中下标为offset的位置开始读入数据，读完后返回读取的字节数

- void close(): 关闭数据流

- int available()： 返回目前可以从数据流中读取的字节数（但实际的读操作所读得的字节数可能大于该返回值）。

- long skip(long i)： 跳过数据流中指定数量的字节不读取，返回值表示实际跳过的字节数。

  

​	对数据流中字节的读取通常是按从头到尾顺序进行的, 若需要进行反方向读取，则需要用到（Push Back操作）

- boolean markSupported()：用于测试数据流是否支持回推操作，当一个数据流支持 mark() 和 reset() 方法时，返回 true，否则返回 false。
- void mark(int readlimit)：用于标记数据流的当前位置，并划出一个缓冲区，其大小至少为指定参数的大小。
- void reset()：将输入流重新定位到对此流最后调用 mark() 方法时的位置。



### 常见子类

- ByteArrayInputStream：字节数组输入流，该类的功能就是从字节数组 byte[] 中进行以字节为单位的读取，也就是将资源文件都以字节形式存入到该类中的字节数组中去，我们拿数据也是从这个字节数组中拿。
- PipedInputStream：管道字节输入流，它和 PipedOutputStream 一起使用，能实现多线程间的管道通信。
- FilterInputStream：装饰者模式中充当装饰者的角色，具体的装饰者都要继承它，所以在该类的子类下都是用来装饰别的流的，也就是处理类。
- BufferedInputStream：缓冲流，对处理流进行装饰、增强，内部会有一个缓冲区，用来存放字节，每次都是将缓冲区存满然后发送，而不是一个字节或两个字节这样发送，效率更高。
- DataInputStream：数据输入流，用来装饰其他输入流，它允许通过数据流来读写Java基本类型。
- FileInputStream：文件输入流，通常用于对文件进行读取操作。
- File：对指定目录的文件进行操作。
- ObjectInputStream：对象输入流，用来提供对“基本数据或对象”的持久存储。通俗点讲，就是能直接传输Java对象（序列化、反序列化用）。



## 字节输出流OutputStream（抽象类）

java.io 包下所有字节输出流大多是从抽象类 OutputStream 继承而来的。OutputStream 提供的主要数据操作方法：

- void write(int i)：将字节 i 写入到数据流中，它只输出所读入参数的最低 8 位，该方法是抽象方法，需要在其输出流子类中加以实现，然后才能使用。
- void write(byte[] b)：将数组 b 中的全部 b.length 个字节写入数据流。
- void write(byte[] b, int off, int len)：将数组 b 中从下标 off 开始的 len 个字节写入数据流。元素 b[off] 是此操作写入的第一个字节，b[off + len - 1] 是此操作写入的最后一个字节。
- void close()：关闭输出流。
- void flush()：刷新此输出流并强制写出所有缓冲的输出字节。

为了加快数据传输速度，提高数据输出效率，有时输出数据流会在提交数据之前把所要输出的数据先暂时保存在内存缓冲区中，然后成批进行输出，每次传输过程都以某特定数据长度为单位进行传输，在这种方式下，数据的末尾一般都会有一部分数据由于数量不够一个批次，而存留在缓冲区里，调用 flush() 方法可以将这部分数据强制提交。

![img](https://pics3.baidu.com/feed/9a504fc2d5628535e8ccc88193203ec0a6ef63b2.jpeg?token=2793c22f9e990e68df32bbb7a1ff9dc6&s=18A87C32DF585CCA02F5D1CA0000E0B2)

## 字符流操作

同其他程序设计语言使用ASCII字符集不同，Java使用Unicode字符集来表示字符串和字符。ASCII字符集以一个字节（8bit）表示一个字符，可以认为一个字符就是一个字节（byte）。但Java使用的Unicode是一种大字符集，用两个字节（16bit）来表示一个字符，这时字节与字符就不再相同。为了实现与其他程序语言及不同平台的交互，Java提供一种新的数据流处理方案，称作读者（Reader）和写者（Writer）。

## 字符流输入流Reader

是所有的字符输入流的父类，Reader是一个抽象类

- CharReader和SringReader是两种基本的介质流，它们分别将Char数组、String中读取数据。PipedReader 是从与其它线程共用的管道中读取数据。
- BufferedReader很明显是一个装饰器，它和其他子类负责装饰其他Reader对象。
- FilterReader是所有自定义具体装饰流的父类，其子类PushBackReader对Reader对象进行装饰，会增加一个行号。
- InputStreamReader是其中最重要的一个，用来在字节输入流和字符输入流之间作为中介，可以将字节输入流转换为字符输入流。FileReader 可以说是一个达到此功能、常用的工具类，在其源代码中明显使用了将FileInputStream 转变为Reader 的方法。

Reader 中各个类的用途和使用方法基本和InputStream 中的类使用一致。



## 字符流输出流Writer

同理，是所有字符输出流的父类，是一个抽象类

- CharWriter、StringWriter 是两种基本的介质流，它们分别向Char 数组、String 中写入数据。
- PipedWriter 是向与其它线程共用的管道中写入数据
- BufferedWriter 是一个装饰器为Writer 提供缓冲功能。
- PrintWriter 和PrintStream 极其类似，功能和使用也非常相似。
- OutputStreamWriter是其中最重要的一个，用来在字节输出流和字符输出流之间作为中介，可以将字节输出流转换为字符输出流。FileWriter 可以说是一个达到此功能、常用的工具类，在其源代码中明显使用了将OutputStream转变为Writer 的方法。



字节流一般用来处理图像，视频，以及PPT，Word类型的文件。字符流一般用于处理纯文本类型的文件，如TXT文件等。字节流可以用来处理纯文本文件，但是字符流不能用于处理图像视频等非文本类型的文件。



## 序列化与反序列化

https://www.cnblogs.com/xdp-gacl/p/3777987.html

**把对象转换为字节序列的过程称为对象的序列化**。

**把字节序列恢复为对象的过程称为对象的反序列化**。



对象序列化的两个主要用途：

- 把对象的字节序列永久存放到硬盘中，通常是保存在一个文件
- 在网络上传送对象的字节序列

把Java对象转换为字节序列的过程称为对象的序列化，**也就是将对象写入到IO流中**。序列化是为了解决在对对象流进行读写操作时所引发的问题。序列化机制允许将实现序列化的Java对象转换位字节序列，这些字节序列可以保存在磁盘上，或通过网络传输，以达到以后恢复成原来的对象。序列化机制使得对象可以脱离程序的运行而独立存在。

只有实现了Serializable和Externalizable接口的类的对象才能被序列化。Externalizable接口继承自 Serializable接口，实现Externalizable接口的类完全由自身来控制序列化的行为，而仅实现Serializable接口的类可以 采用默认的序列化方式 。

下面介绍的是Serializable的默认序列化方式

### **序列化操作**

要对一个对象序列化，这个对象就需要实现Serializable接口，若这个对象中有一个变量是另一个对象的引用，则引用的对象也需要实现Serializable接口，这个过程是递归的。

**Serializable接口中没有定义任何方法，只是作为一个标记来指示实现该接口的类可以进行序列化。**

要实现序列化，只需两步即可：

- 步骤一：创建一个ObjectOutputStream输出流；
- 步骤二：调用ObjectOutputStream对象的 writeObject 方法输出可序列化对象。



序列化只能保存对象的非静态成员变量，而不能保存任何成员方法和静态成员变量，并且保存的只是变量的值，变量的修饰符对序列化没有影响。

有一些对象类不具有可持久化性，因为其数据的特性决定了它会经常变化，其状态只是瞬时的，这样的对象是无法保存去状态的，如Thread对象或流对象。对于这样的成员变量，必须用 transient 关键字标明，否则编译器将报错。**任何用 transient 关键字标明的成员变量，都不会被保存。**

### **反序列化操作**

反序列 是从IO流中恢复对象

反序列化也只需两步即可完成：

- 步骤一：创建 ObjectInputStream 输入流
- 步骤二：调用ObjectInputStream对象的readObject()得到序列化的对象。



### **序列化版本号serialVersionUID**

序列化和反序列化必然包含着class文件，但随着项目的升级，class也会变化，于是采用了版本号serialVersionUID来保证其兼容性

java序列化提供了一个private static final long serialVersionUID 的序列化版本号，只有版本号相同，即使更改了序列化属性，对象也可以正确被反序列化回来。

如果反序列化使用的class的版本号与序列化时使用的不一致，反序列化会报InvalidClassException异常



序列化版本号可自由指定，如果不指定，JVM会根据类信息自己计算一个版本号，这样随着class的升级，就无法正确反序列化；不指定版本号另一个明显隐患是，

不利于jvm间的移植，可能class文件没有更改，

但不同jvm可能计算的规则不一样，这样也会导致无法反序列化。

### **序列化场景**

- 所有需要网络传输的对象都需要实现序列化接口，通过建议所有的javaBean都实现Serializable接口。
- 对象的类名、实例变量（包括基本类型，数组，对其他对象的引用）都会被序列化；
- 方法、类变量、transient实例变量都不会被序列化。如果想让某个变量不被序列化，使用transient修饰。
- 序列化对象的引用类型成员变量，也必须是可序列化的，否则，会报错。
- 反序列化时必须有序列化对象的class文件。
- 当通过文件、网络来读取序列化后的对象时，必须按照实际写入的顺序读取。
- 单例类序列化，需要重写readResolve()方法；否则会破坏单例原则。
- 同一对象序列化多次，只有第一次序列化为二进制流，以后都只是保存序列化编号，不会重复序列化。
- 建议所有可序列化的类加上serialVersionUID 版本号，方便项目升级。