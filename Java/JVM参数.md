# JVM

## 常量池

### 1. 全局字符串池

string pool，在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到string pool中（**记住：string pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放的**）。

在每个JVM中只有一份，存放的是字符串常量和引用值。



### 2. class文件常量池

编译的时候每个class都有的，编译阶段存放编译器生成的**字面量（Literal）和符号引用（Symbolic References）**。

1、字面量就是我们所说的常量概念，如文本字符串、被声明为final的常量值等。

2、符号引用是一组符号来描述所引用的目标：

- 类和接口的全限定名
- 字段的名称和描述符
- 方法的名称和描述符



### 3. 运行时常量池

在类加载完成之后，将每个class常量池中的符号引用值转存到运行时常量池中，也就是说，每个class都有一个运行时常量池，类在解析之后，将符号引用替换成直接引用，与全局常量池中的引用值保持一致。



1.布尔类型参数值

`-XX:+<option> '+'`表示启用该选项  
`-XX:-<option> '-'`表示关闭该选项  

2.数字类型参数值：

`-XX:<option>=<number>` 给选项设置一个数字类型值，可跟随单位，例如：'m'或'M'表示兆字节;'k'或'K'千字节;'g'或'G'千兆字节。32K与32768是相同大小的。

3.字符串类型参数值：

`-XX:<option>=<string>` 给选项设置一个字符串类型值，通常用于指定一个文件、路径或一系列命令列表。 例如：-XX:HeapDumpPath=./dump.core


1. 堆设置
2. -Xms:初始堆大小，默认物理内存的1/64
3. -Xmx:最大堆大小，默认物理内存的1/4，设置与-Xms一样
2. -XX:NewSize=n:设置年轻代大小
3. -XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
6.  -XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
7. -XX:MaxPermSize=n:设置持久代大小


2. 收集器设置
3. -XX:+UseSerialGC:设置串行收集器o -XX:+UseParallelGC:设置并行收集器
4. -XX:+UseParalledlOldGC:设置并行年老代收集器
5. -XX:+UseConcMarkSweepGC:设置并发收集器


3. 垃圾回收统计信息

-XX:+PrintGC

-XX:+PrintGCDetails

-XX:+PrintGCTimeStamps

-Xloggc:filename


## 内存泄露

程序在申请内存后，无法释放已申请的内存空间，一次内存泄露危害可以忽略，但内存泄露堆积后果很严重，无论多少内存,迟早会被占光，广义并通俗的说，就是：不再会被使用的对象或者变量占用的**内存不能被回收**，就是内存泄露。







1. Java heap space
   当堆内存（Heap Space）没有足够空间存放新创建的对象时，就会抛出 java.lang.OutOfMemoryError: Java heap space 错误。（提示：根据实际生产经验，可以对程序日志中的 OutOfMemoryError 配置关键字告警，一经发现，立即处理）。

原因分析
Java heap space 错误产生的常见原因可以分为以下几类：

请求创建一个超大对象，通常是一个大数组。
超出预期的访问量/数据量，通常是上游系统请求流量飙升，常见于各类促销/秒杀活动，可以结合业务流量指标排查是否有尖状峰值。
过度使用终结器（Finalizer），该对象没有立即被 GC。
内存泄漏（Memory Leak），大量对象引用没有释放，JVM 无法对其自动回收，常见于使用了 File 等资源没有回收。
解决方案
针对大部分情况，通常只需要通过 -Xmx 参数调高 JVM 堆内存空间即可。如果仍然没有解决，可以参考以下情况做进一步处理：

如果是超大对象，可以检查其合理性，比如是否一次性查询了数据库全部结果，而没有做结果数限制。
如果是业务峰值压力，可以考虑添加机器资源，或者做限流降级。
如果是内存泄漏，需要找到持有的对象，修改代码设计，比如关闭没有释放的连接。

2. GC overhead limit exceeded
   当 Java 进程花费 98% 以上的时间执行 GC，但只恢复了不到 2% 的内存，且该动作连续重复了 5 次，就会抛出 java.lang.OutOfMemoryError:GC overhead limit exceeded 错误。简单地说，就是应用程序已经基本耗尽了所有可用内存， GC 也无法回收。

此类问题的原因与解决方案跟 Java heap space 非常类似，可以参考上文。

3. Permgen space
   该错误表示永久代（Permanent Generation）已用满，通常是因为加载的 class 数目太多或体积太大。

原因分析
永久代存储对象主要包括以下几类：

加载/缓存到内存中的 class 定义，包括类的名称，字段，方法和字节码；
常量池；
对象数组/类型数组所关联的 class；
JIT （即时编译）编译器优化后的 class 信息。
PermGen 的使用量与加载到内存的 class 的数量/大小正相关。

解决方案
根据 Permgen space 报错的时机，可以采用不同的解决方案，如下所示：

程序启动报错，修改 -XX:MaxPermSize 启动参数，调大永久代空间。
应用重新部署时报错，很可能是没有应用没有重启，导致加载了多份 class 信息，只需重启 JVM 即可解决。
运行时报错，应用程序可能会动态创建大量 class，而这些 class 的生命周期很短暂，但是 JVM 默认不会卸载 class，可以设置 -XX:+CMSClassUnloadingEnabled 和 -XX:+UseConcMarkSweepGC 这两个参数允许 JVM 卸载 class。
如果上述方法无法解决，可以通过 jmap 命令 dump 内存对象 jmap -dump:format=b,file=dump.hprof <process-id> ，然后利用 Eclipse MAT 功能逐一分析开销最大的 classloader 和重复 class。

4. Metaspace
   JDK 1.8 使用 Metaspace 替换了永久代（Permanent Generation），该错误表示 Metaspace 已被用满，通常是因为加载的 class 数目太多或体积太大。

此类问题的原因与解决方法跟 Permgen space 非常类似，可以参考上文。需要特别注意的是调整 Metaspace 空间大小的启动参数为 -XX:MaxMetaspaceSize。

5. Unable to create new native thread
   每个 Java 线程都需要占用一定的内存空间，当 JVM 向底层操作系统请求创建一个新的 native 线程时，如果没有足够的资源分配就会报此类错误。

原因分析
JVM 向 OS 请求创建 native 线程失败，就会抛出 Unable to create new native thread，常见的原因包括以下几类：

线程数超过操作系统最大线程数 ulimit 限制。
线程数超过 kernel.pid_max（只能重启）。
native 内存不足。
该问题发生的常见过程主要包括以下几步：

JVM 内部的应用程序请求创建一个新的 Java 线程；
JVM native 方法代理了该次请求，并向操作系统请求创建一个 native 线程；
操作系统尝试创建一个新的 native 线程，并为其分配内存；
如果操作系统的虚拟内存已耗尽，或是受到 32 位进程的地址空间限制，操作系统就会拒绝本次 native 内存分配；
JVM 将抛出 java.lang.OutOfMemoryError: Unable to create new native thread 错误。
解决方案
升级配置，为机器提供更多的内存；
降低 Java Heap Space 大小；
修复应用程序的线程泄漏问题；
限制线程池大小；
使用 -Xss 参数减少线程栈的大小；
调高 OS 层面的线程最大数：执行 ulimia -a 查看最大线程数限制，使用 ulimit -u xxx 调整最大线程数限制。
ulimit -a
.... 省略部分内容 .....
max user processes              (-u) 16384

6. Out of swap space？
   该错误表示所有可用的虚拟内存已被耗尽。虚拟内存（Virtual Memory）由物理内存（Physical Memory）和交换空间（Swap Space）两部分组成。当运行时程序请求的虚拟内存溢出时就会报 Out of swap space? 错误。

原因分析
该错误出现的常见原因包括以下几类：

地址空间不足；
物理内存已耗光；
应用程序的本地内存泄漏（native leak），例如不断申请本地内存，却不释放。
执行 jmap -histo:live <pid> 命令，强制执行 Full GC；如果几次执行后内存明显下降，则基本确认为 Direct ByteBuffer 问题。
解决方案
根据错误原因可以采取如下解决方案：

升级地址空间为 64 bit；
使用 Arthas 检查是否为 Inflater/Deflater 解压缩问题，如果是，则显式调用 end 方法。
Direct ByteBuffer 问题可以通过启动参数 -XX:MaxDirectMemorySize 调低阈值。
升级服务器配置/隔离部署，避免争用。

7. Kill process or sacrifice child
   有一种内核作业（Kernel Job）名为 Out of Memory Killer，它会在可用内存极低的情况下“杀死”（kill）某些进程。OOM Killer 会对所有进程进行打分，然后将评分较低的进程“杀死”，具体的评分规则可以参考 Surviving the Linux OOM Killer。

不同于其他的 OOM 错误，Kill process or sacrifice child 错误不是由 JVM 层面触发的，而是由操作系统层面触发的。

原因分析
默认情况下，Linux 内核允许进程申请的内存总量大于系统可用内存，通过这种“错峰复用”的方式可以更有效的利用系统资源。

然而，这种方式也会无可避免地带来一定的“超卖”风险。例如某些进程持续占用系统内存，然后导致其他进程没有可用内存。此时，系统将自动激活 OOM Killer，寻找评分低的进程，并将其“杀死”，释放内存资源。

解决方案
升级服务器配置/隔离部署，避免争用。
OOM Killer 调优。

8. Requested array size exceeds VM limit
   JVM 限制了数组的最大长度，该错误表示程序请求创建的数组超过最大长度限制。

JVM 在为数组分配内存前，会检查要分配的数据结构在系统中是否可寻址，通常为 Integer.MAX_VALUE - 2。

此类问题比较罕见，通常需要检查代码，确认业务是否需要创建如此大的数组，是否可以拆分为多个块，分批执行。

9. Direct buffer memory
   Java 允许应用程序通过 Direct ByteBuffer 直接访问堆外内存，许多高性能程序通过 Direct ByteBuffer 结合内存映射文件（Memory Mapped File）实现高速 IO。

原因分析
Direct ByteBuffer 的默认大小为 64 MB，一旦使用超出限制，就会抛出 Direct buffer memory 错误。

解决方案
Java 只能通过 ByteBuffer.allocateDirect 方法使用 Direct ByteBuffer，因此，可以通过 Arthas 等在线诊断工具拦截该方法进行排查。
检查是否直接或间接使用了 NIO，如 netty，jetty 等。
通过启动参数 -XX:MaxDirectMemorySize 调整 Direct ByteBuffer 的上限值。
检查 JVM 参数是否有 -XX:+DisableExplicitGC 选项，如果有就去掉，因为该参数会使 System.gc() 失效。
检查堆外内存使用代码，确认是否存在内存泄漏；或者通过反射调用 sun.misc.Cleaner 的 clean() 方法来主动释放被 Direct ByteBuffer 持有的内存空间。
内存容量确实不足，升级配置。
推荐工具&产品
Eclipse Memory Analyzer —— JVM 内存分析工具
ARMS —— 阿里云 APM 产品，支持 OOM 异常关键字告警
Arthas —— Java 在线诊断工具





# 垃圾回收算法

引用计数法(一般不采用)

复制算法

标记清除

标记压缩



#### 年轻代（Young Generation）的回收算法

a)	所有新生成的对象首先都是放在年轻代的。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。
b)	新生代内存按照8:1:1的比例分为一个Eden区和两个survivor(s0,s1)区。一个Eden区，两个Survivor区(一般而言)。大部分对象在Eden区中生成。回收时先将Eden区存活对象复制到一个s0区，然后清空Eden区，当这个s0区也存放满了时，则将Eden区和s0区存活对象复制到另一个s1区，然后清空Eden和这个s0区，此时s0区是空的，然后将s0区和s1区交换，即保持s1区为空， 如此往复。
c)	当s1区不足以存放 Eden和s0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC，也就是新生代、老年代都进行回收。
d)	新生代发生的GC也叫做Minor GC，MinorGC发生频率比较高(不一定等Eden区满了才触发)。

#### 年老代（Old Generation）的回收算法

a)	在年轻代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代中。因此，可以认为年老代中存放的都是一些生命周期较长的对象。
b)	内存比新生代也大很多(大概比例是1:2)，当老年代内存满时触发Major GC即Full GC，Full GC发生频率比较低，老年代对象存活时间比较长，存活率标记高。

循环引用

https://segmentfault.com/a/1190000019910501



## JVM指令集

```
指令码 助记符      说明
0x00 nop         无操作
0x01 aconst_null 将null推送至栈顶
0x02 iconst_m1    将int型-1推送至栈顶
0x03 iconst_0    将int型0推送至栈顶
0x04 iconst_1    将int型1推送至栈顶
0x05 iconst_2    将int型2推送至栈顶
0x06 iconst_3    将int型3推送至栈顶
0x07 iconst_4    将int型4推送至栈顶
0x08 iconst_5    将int型5推送至栈顶
0x09 lconst_0    将long型0推送至栈顶
0x0a lconst_1    将long型1推送至栈顶
0x0b fconst_0    将float型0推送至栈顶
0x0c fconst_1    将float型1推送至栈顶
0x0d fconst_2    将float型2推送至栈顶
0x0e dconst_0    将double型0推送至栈顶
0x0f dconst_1    将double型1推送至栈顶
0x10 bipush    将单字节的常量值(-128~127)推送至栈顶
0x11 sipush    将一个短整型常量值(-32768~32767)推送至栈顶
0x12 ldc    将int, float或String型常量值从常量池中推送至栈顶
0x13 ldc_w    将int, float或String型常量值从常量池中推送至栈顶（宽索引）
0x14 ldc2_w    将long或double型常量值从常量池中推送至栈顶（宽索引）
0x15 iload    将指定的int型本地变量推送至栈顶
0x16 lload    将指定的long型本地变量推送至栈顶
0x17 fload    将指定的float型本地变量推送至栈顶
0x18 dload    将指定的double型本地变量推送至栈顶
0x19 aload    将指定的引用类型本地变量推送至栈顶
0x1a iload_0    将第一个int型本地变量推送至栈顶
0x1b iload_1    将第二个int型本地变量推送至栈顶
0x1c iload_2    将第三个int型本地变量推送至栈顶
0x1d iload_3    将第四个int型本地变量推送至栈顶
0x1e lload_0    将第一个long型本地变量推送至栈顶
0x1f lload_1    将第二个long型本地变量推送至栈顶
0x20 lload_2    将第三个long型本地变量推送至栈顶
0x21 lload_3    将第四个long型本地变量推送至栈顶
0x22 fload_0    将第一个float型本地变量推送至栈顶
0x23 fload_1    将第二个float型本地变量推送至栈顶
0x24 fload_2    将第三个float型本地变量推送至栈顶
0x25 fload_3    将第四个float型本地变量推送至栈顶
0x26 dload_0    将第一个double型本地变量推送至栈顶
0x27 dload_1    将第二个double型本地变量推送至栈顶
0x28 dload_2    将第三个double型本地变量推送至栈顶
0x29 dload_3    将第四个double型本地变量推送至栈顶
0x2a aload_0    将第一个引用类型本地变量推送至栈顶
0x2b aload_1    将第二个引用类型本地变量推送至栈顶
0x2c aload_2    将第三个引用类型本地变量推送至栈顶
0x2d aload_3    将第四个引用类型本地变量推送至栈顶
0x2e iaload    将int型数组指定索引的值推送至栈顶
0x2f laload    将long型数组指定索引的值推送至栈顶
0x30 faload    将float型数组指定索引的值推送至栈顶
0x31 daload    将double型数组指定索引的值推送至栈顶
0x32 aaload    将引用型数组指定索引的值推送至栈顶
0x33 baload    将boolean或byte型数组指定索引的值推送至栈顶
0x34 caload    将char型数组指定索引的值推送至栈顶
0x35 saload    将short型数组指定索引的值推送至栈顶
0x36 istore    将栈顶int型数值存入指定本地变量
0x37 lstore    将栈顶long型数值存入指定本地变量
0x38 fstore    将栈顶float型数值存入指定本地变量
0x39 dstore    将栈顶double型数值存入指定本地变量
0x3a astore    将栈顶引用型数值存入指定本地变量
0x3b istore_0    将栈顶int型数值存入第一个本地变量
0x3c istore_1    将栈顶int型数值存入第二个本地变量
0x3d istore_2    将栈顶int型数值存入第三个本地变量
0x3e istore_3    将栈顶int型数值存入第四个本地变量
0x3f lstore_0    将栈顶long型数值存入第一个本地变量
0x40 lstore_1    将栈顶long型数值存入第二个本地变量
0x41 lstore_2    将栈顶long型数值存入第三个本地变量
0x42 lstore_3    将栈顶long型数值存入第四个本地变量
0x43 fstore_0    将栈顶float型数值存入第一个本地变量
0x44 fstore_1    将栈顶float型数值存入第二个本地变量
0x45 fstore_2    将栈顶float型数值存入第三个本地变量
0x46 fstore_3    将栈顶float型数值存入第四个本地变量
0x47 dstore_0    将栈顶double型数值存入第一个本地变量
0x48 dstore_1    将栈顶double型数值存入第二个本地变量
0x49 dstore_2    将栈顶double型数值存入第三个本地变量
0x4a dstore_3    将栈顶double型数值存入第四个本地变量
0x4b astore_0    将栈顶引用型数值存入第一个本地变量
0x4c astore_1    将栈顶引用型数值存入第二个本地变量
0x4d astore_2    将栈顶引用型数值存入第三个本地变量
0x4e astore_3    将栈顶引用型数值存入第四个本地变量
0x4f iastore    将栈顶int型数值存入指定数组的指定索引位置
0x50 lastore    将栈顶long型数值存入指定数组的指定索引位置
0x51 fastore    将栈顶float型数值存入指定数组的指定索引位置
0x52 dastore    将栈顶double型数值存入指定数组的指定索引位置
0x53 aastore    将栈顶引用型数值存入指定数组的指定索引位置
0x54 bastore    将栈顶boolean或byte型数值存入指定数组的指定索引位置
0x55 castore    将栈顶char型数值存入指定数组的指定索引位置
0x56 sastore    将栈顶short型数值存入指定数组的指定索引位置
0x57 pop     将栈顶数值弹出 (数值不能是long或double类型的)
0x58 pop2    将栈顶的一个（long或double类型的)或两个数值弹出（其它）
0x59 dup     复制栈顶数值并将复制值压入栈顶
0x5a dup_x1    复制栈顶数值并将两个复制值压入栈顶
0x5b dup_x2    复制栈顶数值并将三个（或两个）复制值压入栈顶
0x5c dup2    复制栈顶一个（long或double类型的)或两个（其它）数值并将复制值压入栈顶
0x5d dup2_x1    <待补充>
0x5e dup2_x2    <待补充>
0x5f swap    将栈最顶端的两个数值互换(数值不能是long或double类型的)
0x60 iadd    将栈顶两int型数值相加并将结果压入栈顶
0x61 ladd    将栈顶两long型数值相加并将结果压入栈顶
0x62 fadd    将栈顶两float型数值相加并将结果压入栈顶
0x63 dadd    将栈顶两double型数值相加并将结果压入栈顶
0x64 isub    将栈顶两int型数值相减并将结果压入栈顶
0x65 lsub    将栈顶两long型数值相减并将结果压入栈顶
0x66 fsub    将栈顶两float型数值相减并将结果压入栈顶
0x67 dsub    将栈顶两double型数值相减并将结果压入栈顶
0x68 imul    将栈顶两int型数值相乘并将结果压入栈顶
0x69 lmul    将栈顶两long型数值相乘并将结果压入栈顶
0x6a fmul    将栈顶两float型数值相乘并将结果压入栈顶
0x6b dmul    将栈顶两double型数值相乘并将结果压入栈顶
0x6c idiv    将栈顶两int型数值相除并将结果压入栈顶
0x6d ldiv    将栈顶两long型数值相除并将结果压入栈顶
0x6e fdiv    将栈顶两float型数值相除并将结果压入栈顶
0x6f ddiv    将栈顶两double型数值相除并将结果压入栈顶
0x70 irem    将栈顶两int型数值作取模运算并将结果压入栈顶
0x71 lrem    将栈顶两long型数值作取模运算并将结果压入栈顶
0x72 frem    将栈顶两float型数值作取模运算并将结果压入栈顶
0x73 drem    将栈顶两double型数值作取模运算并将结果压入栈顶
0x74 ineg    将栈顶int型数值取负并将结果压入栈顶
0x75 lneg    将栈顶long型数值取负并将结果压入栈顶
0x76 fneg    将栈顶float型数值取负并将结果压入栈顶
0x77 dneg    将栈顶double型数值取负并将结果压入栈顶
0x78 ishl    将int型数值左移位指定位数并将结果压入栈顶
0x79 lshl    将long型数值左移位指定位数并将结果压入栈顶
0x7a ishr    将int型数值右（符号）移位指定位数并将结果压入栈顶
0x7b lshr    将long型数值右（符号）移位指定位数并将结果压入栈顶
0x7c iushr    将int型数值右（无符号）移位指定位数并将结果压入栈顶
0x7d lushr    将long型数值右（无符号）移位指定位数并将结果压入栈顶
0x7e iand    将栈顶两int型数值作“按位与”并将结果压入栈顶
0x7f land    将栈顶两long型数值作“按位与”并将结果压入栈顶
0x80 ior     将栈顶两int型数值作“按位或”并将结果压入栈顶
0x81 lor     将栈顶两long型数值作“按位或”并将结果压入栈顶
0x82 ixor    将栈顶两int型数值作“按位异或”并将结果压入栈顶
0x83 lxor    将栈顶两long型数值作“按位异或”并将结果压入栈顶
0x84 iinc    将指定int型变量增加指定值（i++, i--, i+=2）
0x85 i2l     将栈顶int型数值强制转换成long型数值并将结果压入栈顶
0x86 i2f     将栈顶int型数值强制转换成float型数值并将结果压入栈顶
0x87 i2d     将栈顶int型数值强制转换成double型数值并将结果压入栈顶
0x88 l2i     将栈顶long型数值强制转换成int型数值并将结果压入栈顶
0x89 l2f     将栈顶long型数值强制转换成float型数值并将结果压入栈顶
0x8a l2d     将栈顶long型数值强制转换成double型数值并将结果压入栈顶
0x8b f2i     将栈顶float型数值强制转换成int型数值并将结果压入栈顶
0x8c f2l     将栈顶float型数值强制转换成long型数值并将结果压入栈顶
0x8d f2d     将栈顶float型数值强制转换成double型数值并将结果压入栈顶
0x8e d2i     将栈顶double型数值强制转换成int型数值并将结果压入栈顶
0x8f d2l     将栈顶double型数值强制转换成long型数值并将结果压入栈顶
0x90 d2f     将栈顶double型数值强制转换成float型数值并将结果压入栈顶
0x91 i2b     将栈顶int型数值强制转换成byte型数值并将结果压入栈顶
0x92 i2c     将栈顶int型数值强制转换成char型数值并将结果压入栈顶
0x93 i2s     将栈顶int型数值强制转换成short型数值并将结果压入栈顶
0x94 lcmp    比较栈顶两long型数值大小，并将结果（1，0，-1）压入栈顶
0x95 fcmpl    比较栈顶两float型数值大小，并将结果（1，0，-1）压入栈顶；当其中一个数值为NaN时，将-1压入栈顶
0x96 fcmpg    比较栈顶两float型数值大小，并将结果（1，0，-1）压入栈顶；当其中一个数值为NaN时，将1压入栈顶
0x97 dcmpl    比较栈顶两double型数值大小，并将结果（1，0，-1）压入栈顶；当其中一个数值为NaN时，将-1压入栈顶
0x98 dcmpg    比较栈顶两double型数值大小，并将结果（1，0，-1）压入栈顶；当其中一个数值为NaN时，将1压入栈顶
0x99 ifeq    当栈顶int型数值等于0时跳转
0x9a ifne    当栈顶int型数值不等于0时跳转
0x9b iflt    当栈顶int型数值小于0时跳转
0x9c ifge    当栈顶int型数值大于等于0时跳转
0x9d ifgt    当栈顶int型数值大于0时跳转
0x9e ifle    当栈顶int型数值小于等于0时跳转
0x9f if_icmpeq    比较栈顶两int型数值大小，当结果等于0时跳转
0xa0 if_icmpne    比较栈顶两int型数值大小，当结果不等于0时跳转
0xa1 if_icmplt    比较栈顶两int型数值大小，当结果小于0时跳转
0xa2 if_icmpge    比较栈顶两int型数值大小，当结果大于等于0时跳转
0xa3 if_icmpgt    比较栈顶两int型数值大小，当结果大于0时跳转
0xa4 if_icmple    比较栈顶两int型数值大小，当结果小于等于0时跳转
0xa5 if_acmpeq    比较栈顶两引用型数值，当结果相等时跳转
0xa6 if_acmpne    比较栈顶两引用型数值，当结果不相等时跳转
0xa7 goto    无条件跳转
0xa8 jsr     跳转至指定16位offset位置，并将jsr下一条指令地址压入栈顶
0xa9 ret     返回至本地变量指定的index的指令位置（一般与jsr, jsr_w联合使用）
0xaa tableswitch    用于switch条件跳转，case值连续（可变长度指令）
0xab lookupswitch    用于switch条件跳转，case值不连续（可变长度指令）
0xac ireturn    从当前方法返回int
0xad lreturn    从当前方法返回long
0xae freturn    从当前方法返回float
0xaf dreturn    从当前方法返回double
0xb0 areturn    从当前方法返回对象引用
0xb1 return    从当前方法返回void
0xb2 getstatic    获取指定类的静态域，并将其值压入栈顶
0xb3 putstatic    为指定的类的静态域赋值
0xb4 getfield    获取指定类的实例域，并将其值压入栈顶
0xb5 putfield    为指定的类的实例域赋值
0xb6 invokevirtual    调用实例方法
0xb7 invokespecial    调用超类构造方法，实例初始化方法，私有方法
0xb8 invokestatic    调用静态方法
0xb9 invokeinterface 调用接口方法
0xba --
0xbb new     创建一个对象，并将其引用值压入栈顶
0xbc newarray    创建一个指定原始类型（如int, float, char…）的数组，并将其引用值压入栈顶
0xbd anewarray    创建一个引用型（如类，接口，数组）的数组，并将其引用值压入栈顶
0xbe arraylength 获得数组的长度值并压入栈顶
0xbf athrow    将栈顶的异常抛出
0xc0 checkcast    检验类型转换，检验未通过将抛出ClassCastException
0xc1 instanceof 检验对象是否是指定的类的实例，如果是将1压入栈顶，否则将0压入栈顶
0xc2 monitorenter    获得对象的锁，用于同步方法或同步块
0xc3 monitorexit    释放对象的锁，用于同步方法或同步块
0xc4 wide    <待补充>
0xc5 multianewarray 创建指定类型和指定维度的多维数组（执行该指令时，操作栈中必须包含各维度的长度值），并将其引用值压入栈顶
0xc6 ifnull    为null时跳转
0xc7 ifnonnull    不为null时跳转
0xc8 goto_w    无条件跳转
0xc9 jsr_w    跳转至指定32位offset位置，并将jsr_w下一条指令地址压入栈顶
```

## 垃圾回收器种类



![](.\image\Java\GC.jpg)

### G1

`-Xms5m -Xmx5m -XX:+PrintGCDetails  -XX:+UseG1GC`

```java
[GC pause (G1 Evacuation Pause) (young), 0.0018089 secs]
   [Parallel Time: 0.6 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 112.7, Avg: 112.8, Max: 112.8, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.5, Diff: 0.4, Sum: 1.6]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.3, Max: 0.4, Diff: 0.4, Sum: 3.1]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.2]
      [GC Worker Total (ms): Min: 0.5, Avg: 0.5, Max: 0.6, Diff: 0.1, Sum: 4.9]
      [GC Worker End (ms): Min: 113.3, Avg: 113.3, Max: 113.3, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.2 ms]
   [Other: 1.0 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.9 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 3072.0K(3072.0K)->0.0B(2048.0K) Survivors: 0.0B->1024.0K Heap: 3072.0K(6144.0K)->1133.5K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young), 0.0016612 secs]
   [Parallel Time: 0.8 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 171.8, Avg: 171.8, Max: 171.9, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 0.3, Diff: 0.3, Sum: 0.5]
      [Update RS (ms): Min: 0.0, Avg: 0.2, Max: 0.7, Diff: 0.7, Sum: 1.5]
         [Processed Buffers: Min: 0, Avg: 1.3, Max: 4, Diff: 4, Sum: 13]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 1.2]
      [Termination (ms): Min: 0.0, Avg: 0.4, Max: 0.5, Diff: 0.5, Sum: 3.8]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 0.7, Avg: 0.7, Max: 0.8, Diff: 0.1, Sum: 7.1]
      [GC Worker End (ms): Min: 172.5, Avg: 172.5, Max: 172.6, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.8 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.6 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 2048.0K(2048.0K)->0.0B(2048.0K) Survivors: 1024.0K->1024.0K Heap: 3946.1K(6144.0K)->1614.0K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Humongous Allocation) (young) (initial-mark), 0.0010950 secs]
   [Parallel Time: 0.5 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 188.3, Avg: 188.3, Max: 188.4, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.3, Diff: 0.1, Sum: 2.2]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.2]
         [Processed Buffers: Min: 0, Avg: 1.0, Max: 5, Diff: 5, Sum: 10]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
      [Object Copy (ms): Min: 0.1, Avg: 0.1, Max: 0.2, Diff: 0.1, Sum: 1.4]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 0.4, Avg: 0.4, Max: 0.4, Diff: 0.1, Sum: 4.0]
      [GC Worker End (ms): Min: 188.7, Avg: 188.7, Max: 188.7, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.2 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(2048.0K)->0.0B(2048.0K) Survivors: 1024.0K->1024.0K Heap: 2638.0K(6144.0K)->1528.0K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC concurrent-root-region-scan-start]
[GC pause (G1 Humongous Allocation) (young)[GC concurrent-root-region-scan-end, 0.0005756 secs]
[GC concurrent-mark-start]
, 0.0010326 secs]
   [Root Region Scan Waiting: 0.5 ms]
   [Parallel Time: 0.3 ms, GC Workers: 10]
      [GC Worker Start (ms): Min: 190.1, Avg: 190.1, Max: 190.2, Diff: 0.1]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.2]
      [SATB Filtering (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Processed Buffers: Min: 0, Avg: 0.9, Max: 2, Diff: 2, Sum: 9]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.1, Avg: 0.2, Max: 0.2, Diff: 0.1, Sum: 1.5]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.1, Sum: 2.0]
      [GC Worker End (ms): Min: 190.3, Avg: 190.3, Max: 190.3, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(2048.0K)->0.0B(2048.0K) Survivors: 1024.0K->1024.0K Heap: 1528.0K(6144.0K)->1454.8K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure)  1454K->1193K(6144K), 0.0031069 secs]
   [Eden: 0.0B(2048.0K)->0.0B(2048.0K) Survivors: 1024.0K->0.0B Heap: 1454.8K(6144.0K)->1193.2K(6144.0K)], [Metaspace: 3270K->3270K(4480K)]
 [Times: user=0.16 sys=0.00, real=0.01 secs] 
[Full GC (Allocation Failure)  1193K->1145K(6144K), 0.0030986 secs]
   [Eden: 0.0B(2048.0K)->0.0B(2048.0K) Survivors: 0.0B->0.0B Heap: 1193.2K(6144.0K)->1145.1K(6144.0K)], [Metaspace: 3270K->3268K(4480K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC concurrent-mark-abort]
```





### Parallel Scavenge收集器 Parallel Old

`-Xms5m -Xmx5m -XX:+PrintGCDetails  -XX:+UseParallelGC -XX:+UseParallelOldGC`

```java
[GC (Allocation Failure) [PSYoungGen: 256K->192K(1536K)] 1809K->1885K(5632K), 0.0005799 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

[Full GC (Allocation Failure) [PSYoungGen: 192K->0K(1536K)] [ParOldGen: 1693K->1172K(4096K)] 1885K->1172K(5632K), [Metaspace: 3269K->3269K(4480K)], 0.0040003 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

[GC (Allocation Failure) [PSYoungGen: 0K->0K(1536K)] 1172K->1172K(5632K), 0.0012317 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

# OOM
[Full GC (Allocation Failure) [PSYoungGen: 0K->0K(1536K)] [ParOldGen: 1172K->1101K(4096K)] 1172K->1101K(5632K), [Metaspace: 3269K->3267K(4480K)], 0.0078113 secs] [Times: user=0.09 sys=0.00, real=0.01 secs] 

```

参数说明：

[名称：GC前内存占用 -> GC后内存占用 （该区内存总大小）]  GC前堆内存占用 -> GC后堆内存占用 （堆内存总大小）

+ `[GC (Allocation Failure) [PSYoungGen: 256K->192K(1536K)] 1809K->1885K(5632K), 0.0005799 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]`GC类型：Young GC

新生代总内存 / 堆内存 = 1/3；

+ `[Full GC (Allocation Failure) [PSYoungGen: 192K->0K(1536K)] [ParOldGen: 1693K->1172K(4096K)] 1885K->1172K(5632K), [Metaspace: 3269K->3269K(4480K)], 0.0040003 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]`GC类型：Full GC

新生代总内存/老年代总内存 = 1/2；新生代总内存 + 老年代总内存 = 堆内存

### SerialGC

`-Xms5m -Xmx5m -XX:+PrintGCDetails  -XX:+PrintCommandLineFlags -XX:+UseSerialGC`

```java
-XX:InitialHeapSize=5242880 -XX:MaxHeapSize=5242880 -XX:+PrintCommandLineFlags -XX:+PrintGCDetails -XX:-UseLargePagesIndividualAllocation -XX:+UseSerialGC 
[GC (Allocation Failure) [DefNew: 1664K->192K(1856K), 0.0011112 secs] 1664K->718K(5952K), 0.0011399 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
    
[GC (Allocation Failure) [DefNew: 1650K->150K(1856K), 0.0004731 secs][Tenured: 1305K->1440K(4096K), 0.0019278 secs] 2926K->1440K(5952K), [Metaspace: 3270K->3270K(4480K)], 0.0024290 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]

[Full GC (Allocation Failure) [Tenured: 1440K->1190K(4096K), 0.0016928 secs] 1440K->1190K(5952K), [Metaspace: 3270K->3270K(4480K)], 0.0017108 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```



### CMS

`-Xms5m -Xmx5m -XX:+PrintGCDetails  -XX:+UseParNewGC -XX:+UseConcMarkSweepGC`

```
[GC (Allocation Failure) [ParNew: 1088K->128K(1216K), 0.0016975 secs] 1088K->647K(6016K), 0.0017305 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

[GC (Allocation Failure) [ParNew: 581K->55K(1216K), 0.0004477 secs][CMS: 1480K->1178K(4800K), 0.0023755 secs] 2022K->1178K(6016K), [Metaspace: 3270K->3270K(4480K)], 0.0028529 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

[Full GC (Allocation Failure) [CMS: 1178K->1130K(4800K), 0.0020235 secs] 1178K->1130K(6016K), [Metaspace: 3270K->3270K(4480K)], 0.0020465 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

