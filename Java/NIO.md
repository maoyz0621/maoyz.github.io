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
