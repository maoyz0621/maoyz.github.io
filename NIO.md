---

typora-copy-images-to: ./
---

# NIO



## Buffer

1. IntBuffer

2. FloatBuffer

3. CharBuffer

4. DoubleBuffer

5. ShortBuffer

6. LongBuffer

7. **ByteBuffer**



* position 当前可读写的指针，如果是向ByteBuffer对象中写入一个字节，那么就会向position所指向的地址写入这个字节，如果是从ByteBuffer读出一个字节，那么就会读出position所指向的地址读出这个字节，读写完成后，position加1

 * limit 可以读写的边界，当position到达limit时，就表示将ByteBuffer中的内容读完，或者将ByteBuffer写满了

 * mark 对当前position的标记

 * capacity　ByteBuffer的容量

 * flip()  读写交换

   ```java
   public final Buffer flip() {
       limit = position;
       position = 0;
       mark = -1;
       return this;
   }
   ```

 * clear()

   ```java
   public final Buffer clear() {
       position = 0;
       limit = capacity;
       mark = -1;
       return this;
   }
   ```

 * hasRemaining()

   ```java
   public final boolean hasRemaining() {
       return position < limit;
   }
   ```



## Channel（通道）

数据流是双向的．

AbstractSelectableChannel

​    ServerSocketChannel　实现类ServerSocketChannelImpl

​    SocketChannel　实现类SocketChannelImpl

FileChannel

​    FileChannelImpl

- transferFrom()
- transferTo()



1. 当客户端连接时，会通过ServerSocketChannel得到SocketChannel
2. Selector进行监听select(long timeout)，返回有事件发生的通道Channel个数
3. 将SocketChannel注册到Selector上，register(Selector sel, int ops,Object att)，一个Selector可以注册多个SocketChannel
4. 注册之后返回一个SelectionKey，会和Selector关联（Set集合）
5. 进一步得到各个SelectionKey（有事件发生）
6. 在通过SelectionKey反向获取SocketChannel，SelectableChannel channel()
7. 通过得到的Channel，进行业务处理



## Selector（选择器）

+ static Selector open()
+ int select()
+ int select(long timeout)　监控所有的注册通道，当有IO可以操作时，将对应的SelectionKey加入集合中并返回
+ int selectNow()    // 非阻塞
+ Set<SelectionKey> selectedKeys()　从内部集合中获取所有的SelectionKey
+ Selector wakeup()

实现类**SelectorImpl**

## SelectionKey

+ SelectableChannel channel()

+ Selector selector()

+ Object attachment()   获取共享数据

+ SelectionKey interestOps(int ops);　设置或改变监听事件

+ boolean isAcceptable()

+ boolean isConnectable()

+ boolean isReadable()

+ boolean isValid()

+ boolean isWritable()

+ ```java
  OP_READ = 1 << 0
  OP_WRITE = 1 << 2;
  OP_CONNECT = 1 << 3;   表示客户与服务器的连接已经建立成功
  OP_ACCEPT = 1 << 4;    服务器监听到了客户连接，服务器可以接收这个连接了
  ```

实现类**SelectionKeyImpl**



Server端

```java
import org.apache.commons.compress.utils.IOUtils;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;


public class NioServerSocket {

    public static void main(String[] args) throws IOException {
        new NioServerSocket().main0();
    }

    private void main0() throws IOException {
        ServerSocketChannel serverSocketChannel = null;
        Selector selector = null;

        try {
            // 1. 创建serverSocketChannel
            serverSocketChannel = ServerSocketChannel.open();
            // 2. 创建Selector
            selector = Selector.open();
            // 3. 绑定端口，进行监听
            serverSocketChannel.socket().bind(new InetSocketAddress(6666));
            // 4. 设置为非阻塞
            serverSocketChannel.configureBlocking(false);
            // 5. 将serverSocketChannel注册到Selector，关心事件　OP_ACCEPT
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

            // 循环等待客户端连接
            for (; ; ) {
                if (selector.select(1000) == 0) {
                    continue;
                }

                Iterator<SelectionKey> keyIterator = elector.selectedKeys().iterator();
                while (keyIterator.hasNext()) {
                    SelectionKey key = keyIterator.next();
                    // 移除
                    keyIterator.remove();

                    // OP_ACCEPT事件
                    if (key.isAcceptable()) {
                        accept(key, selector);
                    }

                    // OP_READ事件
                    if (key.isReadable()) {
                        readMsg(key);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            IOUtils.closeQuietly(serverSocketChannel);
            IOUtils.closeQuietly(selector);
            if (serverSocketChannel != null) {
                serverSocketChannel.close();
            }
            if (selector != null) {
                selector.close();
            }

        }
    }

    private void readMsg(SelectionKey key) {
        // 反向获取对应的channel
        SocketChannel socketChannel = (SocketChannel) key.channel();
        try {

            // 获取该channel对用的buffer
            ByteBuffer byteBuffer = (ByteBuffer) key.attachment();
            byteBuffer.clear();
            int read = socketChannel.read(byteBuffer);
            if (read == -1) {
                socketChannel.close();
            } else {
                byteBuffer.flip();
                System.out.println("接收来自[" + socketChannel.getRemoteAddress() + "]的消息:" + new String(byteBuffer.array(), 0, byteBuffer.limit()));
            }
        } catch (IOException e) {
            e.printStackTrace();
            try {
                socketChannel.close();
            } catch (IOException e1) {
                e1.printStackTrace();
            }
        }

    }

    private void accept(SelectionKey key, Selector selector) {
        try {
            ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();
            // 生成一个 SocketChannel
            SocketChannel socketChannel = serverSocketChannel.accept();
            socketChannel.configureBlocking(false);
            // 将SocketChannel注册到Selector
            socketChannel.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(1024));
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}
```

Client端

```java
import org.apache.commons.compress.utils.IOUtils;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.StandardCharsets;

public class NioClient {

    public static void main(String[] args) {
        new NioClient().main0();
    }

    private void main0() {
        SocketChannel socketChannel = null;
        Selector selector = null;
        try {
            // 1. 创建SocketChannel
            socketChannel = SocketChannel.open();
            //　2. 设置非阻塞
            socketChannel.configureBlocking(false);

            InetSocketAddress socketAddress = new InetSocketAddress("127.0.0.1", 6666);

            selector = Selector.open();
            // 注册连接事件
            socketChannel.register(selector, SelectionKey.OP_READ);

            //　连接服务器
            if (!socketChannel.connect(socketAddress)) {
                //　如果connect()连接失败，通过此方法完成连接
                while (!socketChannel.finishConnect()) {
                    System.out.println("正在等待连接");
                }
            }

            // 连接成功．发送数据
            ByteBuffer byteBuffer = ByteBuffer.wrap("message ...".getBytes(StandardCharsets.UTF_8));
            //　发送数据
            socketChannel.write(byteBuffer);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            IOUtils.closeQuietly(socketChannel);
            if (socketChannel != null) {
                try {
                    socketChannel.close();
                } catch (IOException e) {

                }
            }
        }
    }
}
```





## 零拷贝

没有CPU拷贝

mmap:　内存映射，将文件映射到**内核**缓冲区，用户可以共享内存空间的数据，可以减少内核空间到用户空间的拷贝次数．适合小数据量读写

3次上下文切换，3次数据拷贝



sendFile:　适合大文件传输

linux2.1提供, 数据不经过用户态，直接从内核缓冲区进入到Socket Buffer；3次上下文切换，1次CPU拷贝，最少2次数据拷贝

linux2.4提供, 数据不经过用户态，避免了从内核缓冲区进入到Socket Buffer，直接拷贝到协议栈；3次上下文切换，最少2次数据拷贝

NIO提供的零拷贝方法：

​	transfer bytes directly from the filesystem cache to the target channel without actually copying them.

​	FileChannel transferTo(long position, long count, WritableByteChannel target)

## Netty

​	NIO框架，事件驱动模型



###　架构模型



### 线程模型

#### 传统阻塞IO

![传统IO](/home/maoyz0621/myWord/maoyz.github.io/传统IO.png)

#### Reactor模式

Reactor模式中的三种角色：

​	1. **Reactor**:　负责监听listen和分配dispatch事件，将I/O事件分派给相应的Handler

​	2. **Acceptor**:　处理客户端新连接，并分派请求到处理器链中

​	3. **Handlers**:　处理非阻塞读/写任务，资源池管理

##### 1) 单Reactor单线程

![单Reactor单线程](/home/maoyz0621/myWord/maoyz.github.io/单Reactor单线程.png)

其中Reactor和Handler处于同一条线程执行，Acceptor可以看作是特殊的Handler．

缺点：

①	当某个Handler阻塞时，所有client的Handler事件都被阻塞，也会导致Server接收不到新的client连接请求(Acceptor阻塞)．

②	仅仅适用于Handler中业务组件能够快速处理的场景

##### 2) 单Reactor多线程

![单Reactor多线程](/home/maoyz0621/myWord/maoyz.github.io/单Reactor多线程.png)

Handler处理器的执行放在线程池中，多线程处理业务；Reactor独立的线程．

在处理业务逻辑，也就是获取到IO的读写事件之后，交由线程池来处理，Handler收到响应后通过send将响应结果返回给客户端。这样可以减小主Reactor的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。

缺点：
 ①	多线程数据共享和访问比较复杂。如果子线程完成业务处理后，把结果传递给主线程reactor进行发送，就会涉及共享数据的互斥和保护机制。
 ②	Reactor承担所有事件的监听和响应，只在主线程中运行，瞬间高并发会成为性能瓶颈。


##### 3) 主从Reactor多线程（Netty基于）



![主从Reactor多线程](/home/maoyz0621/myWord/maoyz.github.io/主从Reactor多线程.png)

**mainReactor**负责监听ServerSocket，用来处理client新连接的建立，将建立的SocketChannel指定注册subReactor其中一个线程上。
**subReactor**维护自己的Selector, 基于mainReactor 注册的socketChannel多路分离IO读写事件，读写网络数据，对业务处理的功能，另其扔给worker线程池来完成。

Netty基于此，做了改进

![](/home/maoyz0621/myWord/maoyz.github.io/Netty.png)

其工作原理

服务端包含1个boss NioEventLoopGroup和1个work NioEventLoopGroup 

boss NioEventLoop循环任务：

1)　轮询Accept事件

2)	处理Accept　IO事件，与clent建立连接，生成NioSocketChannel，并注册到某个work　NioEventLoop的Selector上

3)	处理任务队列中的任务

work NioEventLoop循环任务：

1)	轮寻Read和Write事件；

2)	处理IO事件，在NioSocketChannel可读/可写事件时进行处理；

3)	处理任务队列中的任务

work NioEventLoop处理业务时，使用Pipeline（管道），通过Pipeline可以获取到对应的Channel，Pipeline维护了很多的Handler

![](/home/maoyz0621/图片/Netty具体线程模型
.png)





任务队列中Task：

1　自定义普通任务 TaskQueue

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 都是痛的同一个Thread, nioEventLoopGroup-n
    // 1 用户自定义任务, 队列是TaskQueue
    ctx.channel().eventLoop().execute(() -> {
        try {
            Thread.sleep(5 * 1000);
            ctx.writeAndFlush(Unpooled.copiedBuffer("Hello " + Thread.currentThread().getName() + " => " + ctx.channel().remoteAddress() + ", Welcome 1" + LocalDateTime.now(), CharsetUtil.UTF_8));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });

    // 此时消耗时间是在上一个的基础之上，所以这次是5+6=11s
    ctx.channel().eventLoop().execute(() -> {
        try {
            Thread.sleep(6 * 1000);
            ctx.writeAndFlush(Unpooled.copiedBuffer("Hello " + Thread.currentThread().getName() + " => " + ctx.channel().remoteAddress() + ", Welcome 2" + LocalDateTime.now(), CharsetUtil.UTF_8));

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
}
```



2　自定义定时任务 ScheduleTaskQueue

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 2 用户自定义定时任务,队列是ScheduleTaskQueue
    // 此时消耗时间是在上一个的基础之上，所以这次是5+6+6=17s
    ctx.channel().eventLoop().schedule(() -> {
        try {
            Thread.sleep(6 * 1000);
            ctx.writeAndFlush(Unpooled.copiedBuffer("Hello " + Thread.currentThread().getName() + " => " + ctx.channel().remoteAddress() + ", Welcome 3" + LocalDateTime.now(), CharsetUtil.UTF_8));

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }, 6, TimeUnit.SECONDS);
}
```



3 　非当前Reactor线程调用Channel中的方法





Future-Listener机制




ServerBootstrap

Bootstrap

Future

ChannelFuture

Selector

ChannelPipeline：是Handler的集合

ChannelHandler
ChannelInboundHandler
ChannelInboundHandlerAdapter
ChannelOutboundHandler
ChannelOutboundHandlerAdapter
ChannelDuplexHandler extends ChannelInboundHandlerAdapter implements ChannelOutboundHandler

ChannelOption.SO_BACKLOG  对应TCP/IP协议listen函数中的backlog函数，初始化服务器可连接队列大小
ChannelOption.SO_KEEPALIVE	　保存连接活动状态



EventLoopGroup    一组EventLoop的抽象，一般有多个EventLoop同时工作，每个EventLoop维护一个Selector实例．

NioEventLoopGroup




ChannelPipeline



![](/home/maoyz0621/myWord/maoyz.github.io/ChannelPipeline.png)

ChannelPipeline 中维护的，是一个由 ChannelHandlerContext 组成的双向链表。这个链表的头是 HeadContext, 链表的尾是 TailContext。而无状态的Handler，作为Context的成员，关联在ChannelHandlerContext 中。在对应关系上，每个 ChannelHandlerContext 中仅仅关联着一个 ChannelHandler。



![](/home/maoyz0621/myWord/maoyz.github.io/ChannelHandlerContext.png)

执行流程：

1 每个Channel会绑定一个ChannelPipeline，ChannelPipeline中也会持有Channel的引用

2 ChannelPipeline持有ChannelHandlerContext链路，保留ChannelHandlerContext的头尾节点指针

3 每个ChannelHandlerContext会对应一个ChannelHandler，也就相当于ChannelPipeline持有ChannelHandler链路

4 ChannelHandlerContext同时也会持有ChannelPipeline引用，也就相当于持有Channel引用

5 ChannelHandler链路会根据Handler的类型，分为InBound和OutBound两条链路





Unpooled   非池化，通过readerIndex writerIndex  capacity 将ByteBuf分成3个区域
	

​	0 -- readerIndex　已经读取的区域
​	readerIndex -- writerIndex  可读的区域
​	writerIndex -- capacity  可写的区域





```
readByte() -->  readerIndex()
writeByte() -->  writerIndex()
capacity()
ByteBuf copiedBuffer(CharSequence string, Charset charset)
```





服务器启动源码



接收请求源码



Pipeline源码



ChannelHandler源码



管道　处理器　上下文



Pipeline调用Handler



心跳源码



EventLoop源码



task加入异步线程池源码



   一　TCP 粘包和拆包
   <p>
   为什么会发生TCP粘包、拆包？
   1、应用程序写入的数据大于套接字缓冲区大小，这将会发生拆包。
   2、应用程序写入数据小于套接字缓冲区大小，网卡将应用多次写入的数据发送到网络上，这将会发生粘包。
   3、进行MSS（最大报文长度）大小的TCP分段，当TCP报文长度-TCP头部长度>MSS的时候将发生拆包。
   4、接收方法不及时读取套接字缓冲区数据，这将发生粘包。
   <p>
   二　粘包、拆包解决办法
   <p>
   1、客户端在发送数据包的时候，每个包都固定长度，比如1024个字节大小，如果客户端发送的数据长度不足1024个字节，则通过补充空格的方式补全到指定长度；
   2、客户端在每个包的末尾使用固定的分隔符，例如\r\n，如果一个包被拆分了，则等待下一个包发送过来之后找到其中的\r\n，然后对其拆分后的头部部分与前一个包的剩余部分进行合并，这样就得到了一个完整的包；
   3、将消息分为头部和消息体，在头部中保存有当前整个消息的长度，只有在读取到足够长度的消息之后才算是读到了一个完整的消息；
   4、通过自定义协议进行粘包和拆包的处理。
   <p>
   三　Netty提供的粘包拆包解决方案
   <p>
   1、FixedLengthFrameDecoder
   对于使用固定长度的粘包和拆包场景，可以使用FixedLengthFrameDecoder，该解码一器会每次读取固定长度的消息，如果当前读取到的消息不足指定长度，那么就会等待下一个消息到达后进行补足。
   <p>
   2、LineBasedFrameDecoder与DelimiterBasedFrameDecoder
   对于通过分隔符进行粘包和拆包问题的处理，Netty提供了两个编解码的类，LineBasedFrameDecoder和DelimiterBasedFrameDecoder。
   这里LineBasedFrameDecoder的作用主要是通过换行符，即\n或者\r\n对数据进行处理；
   而DelimiterBasedFrameDecoder的作用则是通过用户指定的分隔符对数据进行粘包和拆包处理。
   <p>
   3、LengthFieldBasedFrameDecoder与LengthFieldPrepender
   LengthFieldBasedFrameDecoder与LengthFieldPrepender需要配合起来使用
   主要思想是在生成的数据包中添加一个长度字段，用于记录当前数据包的长度。LengthFieldBasedFrameDecoder会按照参数指定的包长度偏移量数据对接收到的数据进行解码，从而得到目标消息体数据；
   而LengthFieldPrepender则会在响应的数据前面添加指定的字节数据，这个字节数据中保存了当前消息体的整体字节数据长度。LengthFieldBasedFrameDecoder的解码过程如下图所示：
   <p>
   关于LengthFieldBasedFrameDecoder，这里需要对其构造函数参数进行介绍：
   <p>
   - maxFrameLength：指定了每个包所能传递的最大数据包大小；
   - lengthFieldOffset：指定了长度字段在字节码中的偏移量；
   - lengthFieldLength：指定了长度字段所占用的字节长度；
   - lengthAdjustment：对一些不仅包含有消息头和消息体的数据进行消息头的长度的调整，这样就可以只得到消息体的数据，这里的lengthAdjustment指定的就是消息头的长度；
   - initialBytesToStrip：对于长度字段在消息头中间的情况，可以通过initialBytesToStrip忽略掉消息头以及长度字段占用的字节。
   <p>
   //添加支持粘包、拆包解码器，意义：从头两个字节解析出数据的长度，并且长度不超过1024个字节
   new LengthFieldBasedFrameDecoder(1024, 0, 2, 0, 2))
   //添加支持粘包、拆包编码器，发送的每个数据都在头部增加两个字节表消息长度
   new LengthFieldPrepender(2))