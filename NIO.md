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

![传统IO](../maoyz.github.io/传统IO.png)

#### Reactor模式

Reactor模式中的三种角色：

​	1. **Reactor**:　负责监听listen和分配dispatch事件，将I/O事件分派给相应的Handler

​	2. **Acceptor**:　处理客户端新连接，并分派请求到处理器链中

​	3. **Handlers**:　处理非阻塞读/写任务，资源池管理

##### 1) 单Reactor单线程

![单Reactor单线程](../maoyz.github.io/单Reactor单线程.png)

其中Reactor和Handler处于同一条线程执行，Acceptor可以看作是特殊的Handler．

缺点：

①	当某个Handler阻塞时，所有client的Handler事件都被阻塞，也会导致Server接收不到新的client连接请求(Acceptor阻塞)．

②	仅仅适用于Handler中业务组件能够快速处理的场景

##### 2) 单Reactor多线程

![单Reactor多线程](../maoyz.github.io/单Reactor多线程.png)

Handler处理器的执行放在线程池中，多线程处理业务；Reactor独立的线程．

在处理业务逻辑，也就是获取到IO的读写事件之后，交由线程池来处理，Handler收到响应后通过send将响应结果返回给客户端。这样可以减小主Reactor的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。

缺点：
 ①	多线程数据共享和访问比较复杂。如果子线程完成业务处理后，把结果传递给主线程reactor进行发送，就会涉及共享数据的互斥和保护机制。
 ②	Reactor承担所有事件的监听和响应，只在主线程中运行，瞬间高并发会成为性能瓶颈。


##### 3) 主从Reactor多线程（Netty基于）



![主从Reactor多线程](../maoyz.github.io/主从Reactor多线程.png)

**mainReactor**负责监听ServerSocket，用来处理client新连接的建立，将建立的SocketChannel指定注册subReactor其中一个线程上。
**subReactor**维护自己的Selector, 基于mainReactor 注册的socketChannel多路分离IO读写事件，读写网络数据，对业务处理的功能，另其扔给worker线程池来完成。

Netty基于此，做了改进

![](../maoyz.github.io/Netty.png)

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

![](../maoyz.github.io/Netty具体线程模型
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



![](../maoyz.github.io/ChannelPipeline.png)

ChannelPipeline 中维护的，是一个由 ChannelHandlerContext 组成的双向链表。这个链表的头是 HeadContext, 链表的尾是 TailContext。而无状态的Handler，作为Context的成员，关联在ChannelHandlerContext 中。在对应关系上，每个 ChannelHandlerContext 中仅仅关联着一个 ChannelHandler。



![](../maoyz.github.io/ChannelHandlerContext.png)

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

```java
# NioEventLoop.java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // 1. read() 操作
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}

# io.netty.channel.nio.AbstractNioMessageChannel
@Override
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                // 2. 读取boss线程中NioServerSocketChannel接收到的请求，并把这些请求放入容器中
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }

        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            // 4. channelRead() 方法
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            closed = closeOnReadError(exception);

            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            }
        }
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}

# io.netty.channel.socket.nio.NioServerSocketChannel
@Override
protected int doReadMessages(List<Object> buf) throws Exception {
    // 3. 获取到serverSocketChannel.accept()方法
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}

# io.netty.bootstrap.ServerBootstrap.(ServerBootstrapAcceptor　extends ChannelInboundHandlerAdapter)
// 5. channelRead()方法
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        // 将客户端注册到worker　线程池 ,添加监听器
        // EventLoopGroup childGroup
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}

# io.netty.channel.nio.AbstractNioChannel
@Override
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```

Pipeline源码

```java
# io.netty.channel.AbstractChannelHandlerContext
private AbstractChannelHandlerContext findContextInbound(int mask) {
    AbstractChannelHandlerContext ctx = this;
    EventExecutor currentExecutor = executor();
    do {
        ctx = ctx.next;
    } while (skipContext(ctx, currentExecutor, mask, MASK_ONLY_INBOUND));
    return ctx;
}

private AbstractChannelHandlerContext findContextOutbound(int mask) {
    AbstractChannelHandlerContext ctx = this;
    EventExecutor currentExecutor = executor();
    do {
        ctx = ctx.prev;
    } while (skipContext(ctx, currentExecutor, mask, MASK_ONLY_OUTBOUND));
    return ctx;
}

private static boolean skipContext(
            AbstractChannelHandlerContext ctx, EventExecutor currentExecutor, int mask, int onlyMask) {
        // Ensure we correctly handle MASK_EXCEPTION_CAUGHT which is not included in the MASK_EXCEPTION_CAUGHT
        return (ctx.executionMask & (onlyMask | mask)) == 0 ||
                // We can only skip if the EventExecutor is the same as otherwise we need to ensure we offload
                // everything to preserve ordering.
                //
                // See https://github.com/netty/netty/issues/10067
                (ctx.executor() == currentExecutor && (ctx.executionMask & mask) == 0);
    }
```

ChannelHandler源码

```
ChannelOutboundHandler<I>
ChannelInboundHandler<I>
ChannelHandlerAdapter
```

HandlerContext源码

```
ChannelInboundInvoker<I>
ChannelOutboundInvoker<I>
ChannelHandlerContext<I>
```

管道ChannelPipeline　处理器ChannelHandler　上下文ChannelHandlerContext

1. 任何一个ChannelSocket创建的同时都会创建一个pipeline
2. 调用pipeline的add***方法添加handler时，都会创建一个包装这handler的Context
3. 这些COntext在pipeline中形成双向链表



```java
# io.netty.channel.AbstractChannel
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    # 1. 任何一个ChannelSocket创建的同时都会创建一个pipeline
    pipeline = newChannelPipeline();
}


# io.netty.channel.DefaultChannelPipeline#DefaultChannelPipeline
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    // 异步回调使用
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    // inbound
    tail = new TailContext(this);
    // 既是inbound，也是outbound
    head = new HeadContext(this);

    // 形成双向链表
    head.next = tail;
    tail.prev = head;
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h);
    }

    return this;
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        // 创建Context，每添加一个handler都会创建一个Context
        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventLoop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            callHandlerAddedInEventLoop(newCtx, executor);
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}

// 链表结构 pre <-> newCtx <-> tail
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

Pipeline调用Handler

```java
# io.netty.channel.DefaultChannelPipeline#DefaultChannelPipeline
public final ChannelPipeline fireChannelRead(Object msg) {
    // 入站，从head handler进行处理
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}

# io.netty.channel.AbstractChannelHandlerContext
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}

// ChannelInboundHandler入站
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

心跳源码

public class ReadTimeoutHandler extends IdleStateHandler
public class WriteTimeoutHandler extends ChannelOutboundHandlerAdapter

```java
# io.netty.handler.timeout.IdleStateHandler
private final boolean observeOutput;	// 是否考虑出站慢的情况
private final long readerIdleTimeNanos;
private final long writerIdleTimeNanos;
private final long allIdleTimeNanos;

public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isActive() && ctx.channel().isRegistered()) {
        // channelActive() event has been fired already, which means this.channelActive() will
        // not be invoked. We have to initialize here instead.
        initialize(ctx);
    } else {
        // channelActive() event has not been fired yet.  this.channelActive() will be invoked
        // and initialization will occur there.
    }
}

private void initialize(ChannelHandlerContext ctx) {
    // Avoid the case where destroy() is called before scheduling timeouts.
    // See: https://github.com/netty/netty/issues/143
    switch (state) {
        case 1:
        case 2:
            return;
    }

    state = 1;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    // 调用eventLoop的schedule()方法，将定时任务添加队列中
    if (readerIdleTimeNanos > 0) {
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                                     readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                                     writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                                  allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}

// 初始化监控出站数据属性
private void initOutputChanged(ChannelHandlerContext ctx) {
    if (observeOutput) {
        Channel channel = ctx.channel();
        Unsafe unsafe = channel.unsafe();
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();

        if (buf != null) {
            lastMessageHashCode = System.identityHashCode(buf.current());
            lastPendingWriteBytes = buf.totalPendingWriteBytes();
            lastFlushProgress = buf.currentProgress();
        }
    }
}

AbstractIdleTask
    AllIdleTimeoutTask
    ReaderIdleTimeoutTask
    WriterIdleTimeoutTask
    
private final class ReaderIdleTimeoutTask extends AbstractIdleTask {
    @Override
    protected void run(ChannelHandlerContext ctx) {
        // 1. 得到用户的超时时间
        long nextDelay = readerIdleTimeNanos;
        // 2. 读取操作结束 channelReadComplete
        if (!reading) {
            // 当前时间减去给定时间和最后一次读时间
            nextDelay -= ticksInNanos() - lastReadTime;
        }
					// 触发事件
        if (nextDelay <= 0) {
            // 将任务再次放入队列，时间是刚开始设置的时间，用于做取消操作
            readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);
								// 表示下一次读取不是第一次，firstReaderIdleEvent会在channelRead中设置成true
            boolean first = firstReaderIdleEvent;
            firstReaderIdleEvent = false;

            try {
                // 创建IdleStateEvent类型的读事件，传给用户的userEventTriggered,完成触发事件的操作
                IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // 继续放入队列
            readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}

private final class WriterIdleTimeoutTask extends AbstractIdleTask {

    @Override
    protected void run(ChannelHandlerContext ctx) {

        long lastWriteTime = IdleStateHandler.this.lastWriteTime;
        long nextDelay = writerIdleTimeNanos - (ticksInNanos() - lastWriteTime);
        if (nextDelay <= 0) {
            // Writer is idle - set a new timeout and notify the callback.
            writerIdleTimeout = schedule(ctx, this, writerIdleTimeNanos, TimeUnit.NANOSECONDS);

            boolean first = firstWriterIdleEvent;
            firstWriterIdleEvent = false;

            try {
                // 出站较慢数据的判断
                if (hasOutputChanged(ctx, first)) {
                    return;
                }
										　// 创建IdleStateEvent类型的写事件
                IdleStateEvent event = newIdleStateEvent(IdleState.WRITER_IDLE, first);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // Write occurred before the timeout - set a new timeout with shorter delay.
            writerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}

private final class AllIdleTimeoutTask extends AbstractIdleTask {

    @Override
    protected void run(ChannelHandlerContext ctx) {

        long nextDelay = allIdleTimeNanos;
        if (!reading) {
            nextDelay -= ticksInNanos() - Math.max(lastReadTime, lastWriteTime);
        }
        if (nextDelay <= 0) {
            // Both reader and writer are idle - set a new timeout and
            // notify the callback.
            allIdleTimeout = schedule(ctx, this, allIdleTimeNanos, TimeUnit.NANOSECONDS);

            boolean first = firstAllIdleEvent;
            firstAllIdleEvent = false;

            try {
                if (hasOutputChanged(ctx, first)) {
                    return;
                }

                IdleStateEvent event = newIdleStateEvent(IdleState.ALL_IDLE, first);
                channelIdle(ctx, event);
            } catch (Throwable t) {
                ctx.fireExceptionCaught(t);
            }
        } else {
            // Either read or write occurred before the timeout - set a new
            // timeout with shorter delay.
            allIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
        }
    }
}
```

EventLoop源码

```  java
ScheduledExecutorService 定时任务接口
    SingleThreadEventExecutor　　单线程的线程池
    死循环做3件事：监听端口，处理端口事件，处理队列事件，每个EventLoop可以绑定多个Channel，每个Channel只能由一个EventLoop处理
```

```java
# io.netty.util.concurrent.SingleThreadEventExecutor#execute
@Override
public void execute(Runnable task) {
    if (task == null) {
        throw new NullPointerException("task");
    }

    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && wakesUpForTask(task)) {
        wakeup(inEventLoop);
    }
}
```

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