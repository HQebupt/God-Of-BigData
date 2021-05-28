## 简介

> 这个只能当做工具来查询，而且发现有些地方，jdk1.8的命令已经失效了。
>
> 需要搞懂什么？
>
> 1. 各种命令打印出gc日志的含义是基础
> 2. 如何根据GC的信息，进行调优，有没有方法论？
> 3. 有哪些工具可以帮助
> 4. 有哪些命令分别用在什么场景

运用jvm自带的命令可以方便的在生产监控和打印堆栈的日志信息帮忙我们来定位问题！

虽然jvm调优成熟的工具已经有很多：jconsole、大名鼎鼎的VisualVM，IBM的Memory Analyzer等等，但是在生产环境出现问题的时候，一方面工具的使用会有所限制。

所有的工具几乎都是依赖于jdk的接口和底层的这些命令，研究这些命令的使用也让我们更能了解jvm构成和特性。
Sun JDK监控和故障处理命令有jps jstat jmap jhat jstack jinfo下面做一一介绍

### 1**jps**
JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。

option参数
-l : 输出主类全名或jar路径
-m : 输出JVM启动时传递给main()的参数
-v : 输出JVM启动时显示指定的JVM参数

```bash
$ jps -lmv
 8104 loris-web-http.jar -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true -verbose:gc -Xloggc:/home/loris/loris-jar//logs/loris-web-http-gc.log -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -Dsun.rmi.dgc.server.gcInterval=600000 -Dsun.rmi.dgc.client.gcInterval=600000
18480 loris-test.jar -Xmx3550m -Xms3550m -Xmn2g -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
```

### 2**jstat**
jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

命令格式
jstat [option] LVMID [interval] [count]
参数
[option] : 操作参数
LVMID : 本地虚拟机进程ID
[interval] : 连续输出的时间间隔
[count] : 连续输出的次数

option 参数总览
![9ae139c199be662b76b7860da58f0f07](jvm系列(四)jvm调优-命令大全（jps jstat jmap jhat jstack jinfo）.resources/3358A9D6-B4C7-4831-B26B-AD24973357EB.png)

#### option 参数详解
-class
监视类装载、卸载数量、总空间以及耗费的时间

```
$ jstat -class 27348 1000 10 // 1000ms输出，共10次
Loaded  Bytes  Unloaded  Bytes     Time
 10276 18659.4        0     0.0      14.72
Loaded  Bytes  Unloaded  Bytes     Time
 21257 36337.8     3836  5656.6      27.61
Loaded : 加载class的数量
Bytes : class字节大小
Unloaded : 未加载class的数量
Bytes : 未加载class的字节大小
Time : 加载时间

```
-compiler
输出JIT编译过的方法数量耗时等

```
$ jstat -compiler 1262
Compiled Failed Invalid   Time   FailedType FailedMethod
    7610      8       0    93.74          1 org/springframework/core/annotation/AnnotationUtils postProcessAnnotationAttributes
Compiled : 编译数量
Failed : 编译失败数量
Invalid : 无效数量
Time : 编译耗时
FailedType : 失败类型
FailedMethod : 失败方法的全限定名
```


-gc
垃圾回收堆的行为统计，常用命令

```
$ jstat -gc 1
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
58368.0 58880.0 36632.5  0.0   569344.0 377053.8  689152.0   117122.4  155468.0 145640.3 20096.0 18445.6    979   37.872  46     24.417   62.289
```
C即Capacity 总容量，U即Used 已使用的容量，单位是KB；时间的单位是秒。

各个数据项的含义如下：

- S0C：from 区的大小
- S1C：to 区的大小
- S1U：from 区使用的大小
- S1U：to 去使用的大小
- EC：eden 区的大小
- EU：eden 去使用的大小
- OC：老年代的大小
- OU：老年代使用的大小
- MC：方法区的大小
- MU：方法区使用的大小
- CCSC：压缩类空间大小
- CCUS：压缩类空间使用的大小
- YGC：年轻代垃圾回收的次数
- YGCT：年轻代垃圾回收消耗时间
- FGC：老年代垃圾回收的次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收的总时间

>  可以使用可视化的分析工具来对 CG 日志进行分析，比如 https://gceasy.io/。

```
$ jstat -gc 1262 2000 20
```
这个命令意思就是每隔2000ms输出1262的gc情况，一共输出20次

-gccapacity
同-gc，不过还会输出Java堆各区域使用到的最大、最小空间

```
jstat -gccapacity 1
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC
 87040.0 698880.0 692224.0 124928.0 127488.0 437248.0   175104.0  1398272.0   689152.0   689152.0      0.0 1185792.0 155468.0      0.0 1048576.0  20096.0    997    47
```
NGCMN : 新生代占用的最小空间
NGCMX : 新生代占用的最大空间

NGC：当前新生代包括Eden和Survivor的总容量

OGCMN : 老年代占用的最小空间
OGCMX : 老年代占用的最大空间
OGC：当前年老代的容量 (KB) 与下面的值含义一样。
OC：当前年老代的空间 (KB)
PGCMN : perm占用的最小空间
PGCMX : perm占用的最大空间

-gcutil

同-gc，不过输出的是已使用空间占自己所在区域的总空间的百分比
```
 jstat -gcutil 1
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  56.77  13.87  17.00  93.76  91.95   1012   39.193    47   24.816   64.010
```
-gccause

垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因
```
jstat -gccause 1
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
 59.76   0.00   9.88  17.03  93.77  91.95   1013   39.239    47   24.816   64.056 Allocation Failure   No GC
```
LGCC：最近垃圾回收的原因
GCC：当前垃圾回收的原因

- -gcnew
  统计新生代的行为

```
jstat -gcnew  1
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT
76288.0 81408.0 32093.0    0.0 15  15 81408.0 484352.0 122968.6   1015   39.288
```
TT：Tenuring threshold(提升阈值)
MTT：最大的tenuring threshold
DSS：survivor区域大小 (KB)

- -gcnewcapacity 新生代与其相应的内存空间的统计: 其实和上面的输出信息是一样的。

```
 jstat -gcnewcapacity 1
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC
  87040.0   698880.0   628736.0 232960.0  60928.0 232960.0  62976.0   697856.0   502784.0  1025    47
```
NGC:当前年轻代的容量 (KB)
S0CMX:最大的S0空间 (KB)
S0C:当前S0空间 (KB)
ECMX:最大eden空间 (KB)
EC:当前eden空间 (KB)

- -gcmetacapacity  元数据空间统计

```
jstat -gcmetacapacity 27348
   MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT     GCT
   0.0  1105920.0    64768.0        0.0  1048576.0     7936.0    19     3    1.197    1.759
```

- **MCMN:** 最小元数据容量

- **MCMX：**最大元数据容量

- **MC：**当前元数据空间大小

- **CCSMN：**最小压缩类空间大小

- **CCSMX：**最大压缩类空间大小

- **CCSC：**当前压缩类空间大小

  

- 统计老年代的行为 -gcold

```java
$ jstat -gcold 1
   PC       PU        OC           OU       YGC    FGC    FGCT     GCT   
1048576.0  46561.7   6291456.0     0.0      4      0      0.000    0.242
 jstat -gcold  1
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT
155468.0 145828.6  20096.0  18479.0    689152.0    118660.6   1029    47   24.816   64.496
```
PC MC方法区的容量

PU MU 方法区已经使用的

CCSC CCSU: 压缩类空间的大小

- 统计老年代的大小和空间-gcoldcapacity

```
jstat -gcoldcapacity 1
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT
   175104.0   1398272.0    689152.0    689152.0  1031    47   24.816   64.570
```
##  

-printcompilation

hotspot编译方法统计
```
jstat -printcompilation 1
Compiled  Size  Type Method
   39951     87    1 org/springframework/data/jpa/repository/query/PartTreeJpaQuery getExecution
```
Compiled：被执行的编译任务的数量
Size：方法字节码的字节数（单位：bytes）
Type：编译类型
Method：编译方法的类名和方法名。类名使用”/” 代替 “.” 作为空间分隔符. 方法名是给出类的方法名. 格式是一致于HotSpot - XX:+PrintComplation 选项

### 3 **jmap**
jmap(JVM Memory Map)命令用于生成heap dump文件，如果不使用这个命令，还阔以使用-XX:+HeapDumpOnOutOfMemoryError参数来让虚拟机出现OOM的时候·自动生成dump文件。 jmap不仅能生成dump文件，还阔以查询finalize执行队列、Java堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。

命令格式
jmap [option] LVMID

option参数

dump : 生成堆转储快照
finalizerinfo : 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
heap : 显示Java堆详细信息
histo : 显示堆中对象的统计信息
permstat : to print permanent generation statistics
F : 当-dump没有响应时，强制生成dump快照



- -dump

```
-dump::live,format=b,file=<filename> pid 
```

dump堆到文件,format指定输出格式，live指明是活着的对象,file指定文件名
```
$ jmap -dump:live,format=b,file=dump.hprof 28920
  Dumping heap to /home/xxx/dump.hprof ...
  Heap dump file created
```
>  dump.hprof这个后缀是为了后续可以直接用MAT(Memory Anlysis Tool)打开。

- -finalizerinfo
  打印等待回收对象的信息

```
$ jmap -finalizerinfo 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01
  Number of objects pending for finalization: 0
```
可以看到当前F-QUEUE队列中并没有等待Finalizer线程执行finalizer方法的对象。

- -heap
  打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况,可以用此来判断内存目前的使用情况以及垃圾回收情况

```
$ jmap -heap 28920
  Attaching to process ID 28920, please wait...
  Debugger attached successfully.
  Server compiler detected.
  JVM version is 24.71-b01  

  using thread-local object allocation.
  Parallel GC with 4 thread(s)//GC 方式  

  Heap Configuration: //堆内存初始化配置
     MinHeapFreeRatio = 0 //对应jvm启动参数-XX:MinHeapFreeRatio设置JVM堆最小空闲比率(default 40)
     MaxHeapFreeRatio = 100 //对应jvm启动参数 -XX:MaxHeapFreeRatio设置JVM堆最大空闲比率(default 70)
     MaxHeapSize      = 2082471936 (1986.0MB) //对应jvm启动参数-XX:MaxHeapSize=设置JVM堆的最大大小
     NewSize          = 1310720 (1.25MB)//对应jvm启动参数-XX:NewSize=设置JVM堆的‘新生代’的默认大小
     MaxNewSize       = 17592186044415 MB//对应jvm启动参数-XX:MaxNewSize=设置JVM堆的‘新生代’的最大大小
     OldSize          = 5439488 (5.1875MB)//对应jvm启动参数-XX:OldSize=<value>:设置JVM堆的‘老生代’的大小
     NewRatio         = 2 //对应jvm启动参数-XX:NewRatio=:‘新生代’和‘老生代’的大小比率
     SurvivorRatio    = 8 //对应jvm启动参数-XX:SurvivorRatio=设置年轻代中Eden区与Survivor区的大小比值 
     MaxPermSize      = 85983232 (82.0MB)//对应jvm启动参数-XX:MaxPermSize=<value>:设置JVM堆的‘永生代’的最大大小
     G1HeapRegionSize = 0 (0.0MB)  

  Heap Usage://堆内存使用情况
  PS Young Generation
  Eden Space://Eden区内存分布
     capacity = 33030144 (31.5MB)//Eden区总容量
     used     = 1524040 (1.4534378051757812MB)  //Eden区已使用
     free     = 31506104 (30.04656219482422MB)  //Eden区剩余容量
     4.614088270399305% used //Eden区使用比率
  From Space:  //其中一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  To Space:  //另一个Survivor区的内存分布
     capacity = 5242880 (5.0MB)
     used     = 0 (0.0MB)
     free     = 5242880 (5.0MB)
     0.0% used
  PS Old Generation //当前的Old区内存分布
     capacity = 86507520 (82.5MB)
     used     = 0 (0.0MB)
     free     = 86507520 (82.5MB)
     0.0% used
  670 interned Strings occupying 43720 bytes.
```
可以很清楚的看到Java堆中各个区域目前的情况。

> 因为Perm区在jdk1.8之后就移除heap，所以一般在这里是看不到的。

- -histo
  打印堆的对象统计，包括对象数、内存大小等等 （因为在dump:live前会进行full gc，如果带上live则只统计活对象，因此不加live的堆大小要大于加live堆的大小 ）

```
$ jmap -histo:live 27348 | more
 num     #instances         #bytes  class name
----------------------------------------------
   1:         83613       12012248  <constMethodKlass>
   2:         23868       11450280  [B
   3:         83613       10716064  <methodKlass>
   4:         76287       10412128  [C
   5:          8227        9021176  <constantPoolKlass>
   6:          8227        5830256  <instanceKlassKlass>
   7:          7031        5156480  <constantPoolCacheKlass>
   8:         73627        1767048  java.lang.String
   9:          2260        1348848  <methodDataKlass>
  10:          8856         849296  java.lang.Class
  ....
```
仅仅打印了前10行

xml class name是对象类型，说明如下：

B  byte
C  char
D  double
F  float
I  int
J  long
Z  boolean
[  数组，如[I表示int[]
[L+类名 其他对象

- -F
  强制模式。如果指定的pid没有响应，请使用jmap -dump或jmap -histo选项。此模式下，不支持live子选项。

#### 5 jhat
jhat(JVM Heap Analysis Tool)命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看。

>  注意 因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。

命令格式
jhat [dumpfile]

参数

* -stack false|true 关闭对象分配调用栈跟踪(tracking object allocation call stack)。 如果分配位置信息在堆转储中不可用. 则必须将此标志设置为 false. 默认值为 true.>

* -refs false|true 关闭对象引用跟踪(tracking of references to objects)。 默认值为 true. 默认情况下, 返回的指针是指向其他特定对象的对象,如反向链接或输入引用(referrers or incoming references), 会统计/计算堆中的所有对象。>

* -port port-number 设置 jhat HTTP server 的端口号. 默认值 7000.>

* -exclude exclude-file 指定对象查询时需要排除的数据成员列表文件(a file that lists data members that should be excluded from the reachable objects query)。 例如, 如果文件列列出了 java.lang.String.value , 那么当从某个特定对象 Object o 计算可达的对象列表时, 引用路径涉及 java.lang.String.value 的都会被排除。>

* -baseline exclude-file 指定一个基准堆转储(baseline heap dump)。 在两个 heap dumps 中有相同 object ID 的对象会被标记为不是新的(marked as not being new). 其他对象被标记为新的(new). 在比较两个不同的堆转储时很有用.>

* -debug int 设置 debug 级别. 0 表示不输出调试信息。 值越大则表示输出更详细的 debug 信息.>

* -version 启动后只显示版本信息就退出>

* -J< flag > 因为 jhat 命令实际上会启动一个JVM来执行, 通过 -J 可以在启动JVM时传入一些启动参数. 例如, -J-Xmx512m 则指定运行 jhat 的Java虚拟机使用的最大堆内存为 512 MB. 如果需要使用多个JVM启动参数,则传入多个 -Jxxxxxx.

示例

```
$ jhat -J-Xmx512m dump.hprof
  eading from dump.hprof...
  Dump file created Fri Mar 11 17:13:42 CST 2016
  Snapshot read, resolving...
  Resolving 271678 objects...
  Chasing references, expect 54 dots......................................................
  Eliminating duplicate references......................................................
  Snapshot resolved.
  Started HTTP server on port 7000
  Server is ready.
```

中间的-J-Xmx512m是在dump快照很大的情况下分配512M内存去启动HTTP服务器，运行完之后就可在浏览器打开Http://localhost:7000进行快照分析 堆快照分析主要在最后面的Heap Histogram里，里面根据class列出了dump的时候所有存活对象。

分析同样一个dump快照，MAT需要的额外内存比jhat要小的多的多，所以建议使用MAT来进行分析，当然也看个人偏好。

分析
打开浏览器Http://localhost:7000，该页面提供了几个查询功能可供使用：

```
All classes including platform
Show all members of the rootset
Show instance counts for all classes (including platform)
Show instance counts for all classes (excluding platform)
Show heap histogram
Show finalizer summary
Execute Object Query Language (OQL) query
```
一般查看堆异常情况主要看这个两个部分： Show instance counts for all classes (excluding platform)，平台外的所有对象信息。如下图：
![7f32f8469cc634a9c9a344ee905f9654](jvm系列(四)jvm调优-命令大全（jps jstat jmap jhat jstack jinfo）.resources/E7DB83E0-2344-4149-8603-C606D78AB943.png)
Show heap histogram 以树状图形式展示堆情况。如下图：![80be25d2f1c6b019c28d96c42842fac6](jvm系列(四)jvm调优-命令大全（jps jstat jmap jhat jstack jinfo）.resources/8DE752AB-6C96-4A58-8E01-8970BA3E2014.png)
具体排查时需要结合代码，观察是否大量应该被回收的对象在一直被引用或者是否有占用内存特别大的对象无法被回收。一般情况，会down到客户端用工具来分析

#### 6**jstack**
jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 

命令格式
jstack [option] LVMID
option参数
-F : 当正常输出请求不被响应时，强制输出线程堆栈
-l : 除堆栈外，显示关于锁的附加信息
-m : 如果调用到本地方法的话，可以显示C/C++的堆栈

#### 线程状态

查看线程堆栈信息时可能会看到的**线程的几种状态**：

> NEW,未启动的。不会出现在Dump中。
>
> RUNNABLE,在虚拟机内执行的。
>
> BLOCKED,受阻塞并等待监视器锁。
>
> WATING,无限期等待另一个线程执行特定操作。
>
> TIMED_WATING,有时限的等待另一个线程的特定操作。
>
> TERMINATED,已退出的。

示例

```
$ jstack -l 11494|more
2016-07-28 13:40:04
Full thread dump Java HotSpot(TM) 64-Bit Server VM (24.71-b01 mixed mode):

"Attach Listener" daemon prio=10 tid=0x00007febb0002000 nid=0x6b6f waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
        - None

"http-bio-8005-exec-2" daemon prio=10 tid=0x00007feb94028000 nid=0x7b8c waiting on condition [0x00007fea8f56e000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000000cae09b80> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:104)
        at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:32)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
        - None
      .....
```
这里有一篇文章解释的很好 [分析打印出的文件内容](http://www.hollischuang.com/archives/110)


#### #### 7**jinfo**
jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数。 

> jps -v口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo. 这个可以真正生效的参数是什么。

命令格式
jinfo [option] [args] LVMID
option参数
-flag : 输出指定args参数的值
-flags : 不需要args参数，输出所有JVM参数的值
-sysprops : 输出系统属性，等同于System.getProperties()

```
jinfo -flags 27348 
Attaching to process ID 27348, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.292-b10 # 下面的单位都是Byte
Non-default VM flags: -XX:CICompilerCount=12 -XX:InitialHeapSize=1056964608 -XX:MaxHeapSize=16890462208 -XX:MaxNewSize=5629804544 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=352321536 -XX:OldSize=704643072 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
```