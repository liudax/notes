# Java NIO

## Java NIO 简介

Java NIO（New IO）,是JDK1.4版本开始引入的一个新的IO API，可以代替标准的Java IO API。NIO 和IO有着相同的作用和目的，但使用的方式完全不同，NIO支持面向缓冲区、基于通道的IO操作。NIO以更高效的方式进行文件读写操作。

Java NIO 系统的核心在于：通道（Channel）和缓冲区（Buffer）。通道表示打开到IO设备（比如文件、套接字）的连接。若要使用NIO，需要获取**用于连接设备的通道**以及**用于容纳数据的缓冲区**。然后操作缓冲区对数据进行处理。简而言之，**Channel用于传输，Buffer用于存储。**

**NIO和传统IO的区别**

| IO               | NIO                  |
| ---------------- | -------------------- |
| 面向流（Stream）   | 面向缓冲区（Buffer） |
|阻塞IO（Blocking IO）|非阻塞IO（Non Blocking IO）|
|无                 |选择器（Selectors）tog|

## NIO中重要概念

### 缓冲区Buffer

缓冲区：本质上是一块可以存储数据的内存，就像一个数组，可以保存多个相同类型的数据，被封装成了Buffer对象而已。

缓冲区类型

* ByteBuffer
* CharBuffer
* FloatBuffer
* DoubleBuffer
* IntBuffer
* LongBuffer
* ShortBuffer
* MappedByteBuffer 

#### 缓冲区的基本属性

* **容量（capacity）：**表示Buffer中最大的数据容量，不能为负，并且创建后不能修改。
* **限制（limit）:**第一个不应该读取或写入的数据的索引，即位于limit后的数据不可读写。缓冲区的限制不能为负，并且不能大于容量。
* **位置（position）：**下一个要读取或者写入的数据的索引。缓冲区的位置不能为负，并且不能大于限制。
  * ***标记（mark）与重置（reset）：***标记是一个索引，可以通过Buffer中mark()方法指定一个特定的位置position，之后可以通过调用reset()方法恢复到这个position。
  * ![](.\images\缓冲区操作.png)
    不变式：0 <= mark <=position <=limit <= capacity

#### Buffer常用方法

|      方法       | 描述                                                         |
| :-------------: | :----------------------------------------------------------- |
|   **clear()**   | 从读模式切换到写模式，不会清空数据，但后续写数据会覆盖原来的数据，即使有部分数据没有读，也会被遗忘； |
|   **filp()**    | 将缓冲区从写模式切换到读模式，实质是令limit=position；position=0 |
|    compact()    | 从读数据切换到写模式，数据不会被清空，会将所有未读的数据copy到缓冲区头部，后续写数据不会覆盖，而是在这些数据之后写数据 |
|      get()      | 向缓冲区读数据                                               |
|      put()      | 向缓冲区写数据                                              |
|     mark()      | 对position做出标记，配合reset使用                            |
|     reset()     | 将position置为标记值                                         |
| hashRemaining() | 判断缓冲区中是否还有未读元素                                 |
|   remaining()   | 返回position和limit之间的元素个数                            |
|   rewind()          | 设置position为0，可重复读 |

#### 基本用法

1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用`clear()`方法或者`compact()`方法

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf); //read into buffer.
while (bytesRead != -1) {
  buf.flip();  //切换为读模式
  while(buf.hasRemaining()){
      System.out.print((char) buf.get()); // 一次读一个字节
  }
  buf.clear(); //清空缓冲区，切换为写模式
  bytesRead = inChannel.read(buf);
}
aFile.close();

```



#### 直接缓冲区和非直接缓冲区

![](images\直接缓冲区.png)

![](images\非直接缓冲区.png)

通俗的说，直接缓冲区操作的就是物理内存，非直接缓冲区操作的是jvm的缓冲区，然后复制到物理内存。

* 字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则Java 虚拟机会尽最大努力直接在此缓冲区上执行本机I/O 操作。也就是说，在每次调用基础操作系统的一个本机I/O 操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。 
* 直接字节缓冲区可以通过调用此类的allocateDirect() 工厂方法来创建。此方法返回的缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的本机I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。 
* 直接字节缓冲区还可以通过FileChannel 的map() 方法将文件区域直接映射到内存中来创建。该方法返回MappedByteBuffer。Java 平台的实现有助于通过JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在访问期间或稍后的某个时间导致抛出不确定的异常。 
* 字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其isDirect()方法来确定。提供此方法是为了能够在性能关键型代码中执行显式缓冲区管理。

*FileChannel的transferTo和transferFrom内部使用的就是直接缓冲区*


### 通道Channel

通道：类似于流，但是可以异步读写数据（流只可以同步读写），通道是双向的（流时单项的）。通道的数据总要先读到一个Buffer，或者从一个Buffer写入。即通道只与Buffer进行数据交互。

通道的类型：

* FileChannel 从文件中读写数据。

* DatagramChannel 能通过UDP网络读写数据。

* SocketChannel 能通过TCP网络读写数据。

* ServerSocketChannel 可以监听新进来的TCP连接，像Web服务器那样。对于每一个新进来的连接，都会创建一个SocketChannel。

  FileChannel比较特殊，它可以与缓冲区进行数据交互，不能切换到阻塞模式，套接字通道可以切换到非阻塞模式。

#### 获取方式

获取通道的一种方式对支持通道的对象调用getChannel()方法。支持通道的类如下：

* FileInputStream
* FileOutputStream
* RandomAccessFile
* DatagramSocket
* Socket
* ServerSocket

其他方式是使用Files类的静态方法 newByteChannel()，或者通过通道的静态方法Channel.open()打开并返回指定通道。

#### Channel常用方法

| 方法                     | 描述                                         |
| ------------------------ | -------------------------------------------- |
| read(ByteBuffer dst)     | 从Channel中读取数据到ByteBuffer              |
| read(ByteBuffer[] dsts)  | 将Channel中的数据分散到多个ByteBuffer中      |
| write(ByteBuffer src)    | 将ByteBuffer中的数据写到Channel中            |
| write(ByteBuffer[] srcs) | 将多个ByteBuffer中的数据写到Channel中        |
| position()               | 返回此通道的文件位置                         |
| position(long p)         | 设置此通道的文件位置                         |
| size()                   | 返回此通道的文件的当前大小                   |
| truncate(long s)         | 将此通道的文件截取为给定大小                 |
| force(Boolean metaData)  | 强制所有对此通道的文件更新，写入到存储设备中 |





### Channel和Buffer的简单例子

**文件复制**：

字节直接缓冲区：

```java
 @Test
    public void test() throws Exception{
        RandomAccessFile rafIn = new RandomAccessFile("D:\\斗罗大陆.txt","r");
        FileChannel inChannel = rafIn.getChannel();
        RandomAccessFile rafOut = new RandomAccessFile("D:\\斗罗大陆-复制.txt","rw");
        FileChannel outChannel = rafOut.getChannel();
        ByteBuffer buf = ByteBuffer.allocate(1024);
        while(inChannel.read(buf)!=-1){
            buf.flip();
            outChannel.write(buf);
            buf.clear();
        }
        inChannel.close();
        outChannel.close();
    }
```

MappedByteBuffer :

````java
@Test
public void test() throws IOException{
		
		FileChannel inChannel = FileChannel.open(Paths.get("D:\\斗罗大陆.txt"), StandardOpenOption.READ);
		FileChannel outChannel = FileChannel.open(Paths.get("D:\\斗罗大陆-复制.txt"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
		//内存映射文件
		MappedByteBuffer inMappedBuf = inChannel.map(MapMode.READ_ONLY, 0, inChannel.size());
		MappedByteBuffer outMappedBuf = outChannel.map(MapMode.READ_WRITE, 0, inChannel.size());
		//直接对缓冲区进行数据的读写操作
		byte[] dst = new byte[inMappedBuf.limit()];
		inMappedBuf.get(dst);
		outMappedBuf.put(dst);
		
		inChannel.close();
		outChannel.close();
	}
````

直接利用通道完成复制：

```java
 @Test
    public void test3() throws IOException{
        RandomAccessFile rafIn = new RandomAccessFile("D:\\斗罗大陆.txt","r");
        RandomAccessFile rafOut = new RandomAccessFile("D:\\斗罗大陆-复制.txt","rw");
        FileChannel inChannel =rafIn.getChannel();
        FileChannel outChannel =rafOut.getChannel();
        //inChannel.transferTo(0,inChannel.size(),outChannel);
        outChannel.transferFrom(inChannel,0,inChannel.size());
        inChannel.close();
		outChannel.close();
    }
```







