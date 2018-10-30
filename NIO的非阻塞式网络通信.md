# NIO的非阻塞式网络通信

[TOC]

## 非阻塞体现

说NIO是非阻塞式的，主要指的是套接字通道（ServerSocketChannel 等），FileChannel是不支持非阻塞的。

**传统IO方式**在调用InputStream.read()/buffer.readLine()方法时是阻塞的，它会**一直等到数据到来或缓冲区已满时或超时时才会返回**。同样，在调用ServerSocket.accept()方法也是阻塞的。

**NIO方式**中， selector.select()方法在没有事件到达时会一直阻塞。但在读写数据时与传统IO有区别，它是在selector上**注册读、写等事件，服务器处理线程轮询selector中的所有事件，当有感兴趣的事件时才会处理**，并不会像传统IO那样，在InputStream.read()/buffer.readLine()时会阻塞。**Selector就是以事件为驱动的。**

## 选择器Selector

* 选择器（Selector）是SelectableChannel对象的多路复用器，Selector可以统计监控多个SelectableChannel的IO状况，也就是说利用Selector可以用单线程管理多个Channel，Selector是非阻塞IO的核心。
* 关于SelectableChannel的类继承图

![](images\SelectableChannel继承图.png)



## Selector应用

创建Selector：通过调用Selector.open()创建。

```java
Selector selector = Selector.open();
```

向选择器注册事件：SelectableChannel.register(Selector sel,int ops)

```java
Socket socket = new Socket(InetAddress.getByName(127.0.0.1),9999);
SocketChannel channel = socket.getChannel();
Selector selector = Selector.open();
channel.configureBlocking(false);//设置为非阻塞
SelectionKey key = channel.register(selector,SelectionKey.OP_READ);
```

## SelectionKey



在通道注册选择器时，第二个参数ops可通过SelectionKey的四个常量指定：

* 读：SelectionKey.OP_READ（1 << 0）
* 写：SelectionKey.OP_WRITE（1 << 2）
* 连接：SelectionKey.OP_CONNECT（1 << 3）
* 接收：SelectionKey.OP_ACCEPT（1 << 4）

若注册时不止监听一个时间，则可以使用”位或“操作连接符。

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_READ.OP_WRITE;
```





## ***服务器代码:***

```java
package nio;
import java.io.ByteArrayOutputStream;
import java.net.InetSocketAddress;
import java.nio.Buffer;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
    public static void start() throws Exception{
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(9999));
        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);
        while (true){
             int keyNum = selector.select();//没有新事件则阻塞
             if(keyNum<=0){
                 continue;
             }
            Set<SelectionKey> keys =  selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();
            while(it.hasNext()){
                SelectionKey key = it.next();
                if(key.isAcceptable()){
                    ServerSocketChannel ssc = (ServerSocketChannel)key.channel();
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    sc.register(selector,SelectionKey.OP_READ);
                }else if(key.isReadable()){
                    SocketChannel channel = (SocketChannel)key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(1024);
                    ByteArrayOutputStream bos = new ByteArrayOutputStream();
                    int temp;
                    while((temp =channel.read(buf))>0){//这里必须大于0，因为通道不会关闭
                        buf.flip();
                        bos.write(buf.array());
                        buf.clear();
                    }
                    byte[] bytes = bos.toByteArray();
                    System.out.println(new String(bytes,0,bytes.length,"UTF-8"));
                    channel.shutdownInput();
                    channel.register(selector,SelectionKey.OP_WRITE);
                }else if (key.isWritable()){
                    SocketChannel channel = (SocketChannel)key.channel();
                    String httpResponse = "HTTP/1.1 200 OK\r\n" +
                            "Content-Length: 38\r\n" +
                            "Content-Type: text/html\r\n" +
                            "\r\n" +
                            "<html><body>Hello World!</body></html>";
                    ByteBuffer buf = ByteBuffer.allocate(1024);
                    buf.put(httpResponse.getBytes());
                    buf.flip();
                    channel.write(buf);
                    channel.shutdownOutput();
                    channel.close();
                }
                it.remove();
            }

        }
    }
    public static void main(String[] args) throws Exception {
        start();
    }
}
```

## ***客户端代码:***

```java
package nio;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class Client {
    public static void main(String[] args) throws IOException {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1",9999));
        socketChannel.configureBlocking(false);
        String msg ="GET / HTTP/1.1\n" +
                "Host: localhost:9999\n" +
                "Connection: keep-alive\n" +
                "Cache-Control: max-age=0\n" +
                "Upgrade-Insecure-Requests: 1\n" +
                "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.81 Safari/537.36\n" +
                "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8\n" +
                "Accept-Encoding: gzip, deflate, br\n" +
                "Accept-Language: zh-CN,zh;q=0.9\n" +
                "Cookie: olfsk=olfsk18516141882915793; hblid=KYIpjgtKd8pJ8rAA3m39N0H00JEbaRoB; checked=true; username=51180002; Webstorm-b7b4ef1b=f706f9f3-c729-4d06-94a1-c8f4a971fbed";
        byte[] bytes = msg.getBytes("UTF-8");
        ByteBuffer buf = ByteBuffer.allocate(bytes.length);
        buf.put(msg.getBytes());
        buf.flip();
        socketChannel.write(buf);
        socketChannel.shutdownOutput();
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int len;
        while(true){
            buf.clear();
            len = socketChannel.read(buf);
            if(len == -1){
                break;
            }
            buf.flip();
            while(buf.hasRemaining()){
                baos.write(buf.get());
            }
        }
        System.out.println("客户端收到："+new String(baos.toByteArray()));
        socketChannel.close();
    }
}
```

