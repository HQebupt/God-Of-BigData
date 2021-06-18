jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。

> **jstack命令主要用来查看Java线程的调用堆栈的，可以用来分析线程问题（如死锁）。**



### 线程状态

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

### Monitor

在多线程的 JAVA程序中，实现线程之间的同步，就要说说 Monitor。 **Monitor是 Java中用以实现线程之间的互斥与协作的主要手段**，它可以看成是对象或者 Class的锁。每一个对象都有，也仅有一个 monitor。下 面这个图，描述了线程和 Monitor之间关系，以 及线程的状态转换图：

![thread](Jstack分析.assets/thread-1622168647643.bmp)

**进入区(Entrt Set)**:表示线程通过synchronized要求获取对象的锁。如果对象未被锁住,则进入拥有者;否则则在进入区等待。一旦对象锁被其他线程释放,立即参与竞争。

**拥有者(The Owner)**:表示某一线程成功竞争到对象锁。

**等待区(Wait Set)**:表示线程通过对象的wait方法,释放对象的锁,并在等待区等待被唤醒。

从图中可以看出，一个 Monitor在某个时刻，只能被一个线程拥有，该线程就是 `“Active Thread”`，而其它线程都是 `“Waiting Thread”`，分别在两个队列 `“ Entry Set”`和 `“Wait Set”`里面等候。在 `“Entry Set”`中等待的线程状态是 `“Waiting for monitor entry”`，而在 `“Wait Set”`中等待的线程状态是 `“in Object.wait()”`。 先看 “Entry Set”里面的线程。我们称被 synchronized保护起来的代码段为临界区。当一个线程申请进入临界区时，它就进入了 “Entry Set”队列。对应的 code就像：

```java
synchronized(obj) {
.........

}
```

### 调用修饰

表示线程在方法调用时,额外的重要的操作。线程Dump分析的重要信息。修饰上方的方法调用。

> locked <地址> 目标：使用synchronized申请对象锁成功,监视器的拥有者。
>
> waiting to lock <地址> 目标：使用synchronized申请对象锁未成功,在进入区等待。
>
> waiting on <地址> 目标：使用synchronized申请对象锁成功后,释放锁在等待区等待。
>
> parking to wait for <地址> 目标

**locked**

```
at oracle.jdbc.driver.PhysicalConnection.prepareStatement
- locked <0x00002aab63bf7f58> (a oracle.jdbc.driver.T4CConnection)
at oracle.jdbc.driver.PhysicalConnection.prepareStatement
- locked <0x00002aab63bf7f58> (a oracle.jdbc.driver.T4CConnection)
at com.jiuqi.dna.core.internal.db.datasource.PooledConnection.prepareStatement
```

通过synchronized关键字,成功获取到了对象的锁,成为监视器的拥有者,在临界区内操作。对象锁是可以线程重入的。

**waiting to lock**

```
at com.jiuqi.dna.core.impl.CacheHolder.isVisibleIn(CacheHolder.java:165)
- waiting to lock <0x0000000097ba9aa8> (a CacheHolder)
at com.jiuqi.dna.core.impl.CacheGroup$Index.findHolder
at com.jiuqi.dna.core.impl.ContextImpl.find
at com.jiuqi.dna.bap.basedata.common.util.BaseDataCenter.findInfo
```

通过synchronized关键字,没有获取到了对象的锁,线程在监视器的进入区等待。在调用栈顶出现,线程状态为Blocked。

**waiting on**

```
at java.lang.Object.wait(Native Method)
- waiting on <0x00000000da2defb0> (a WorkingThread)
at com.jiuqi.dna.core.impl.WorkingManager.getWorkToDo
- locked <0x00000000da2defb0> (a WorkingThread)
at com.jiuqi.dna.core.impl.WorkingThread.run
```

通过synchronized关键字,成功获取到了对象的锁后,调用了wait方法,进入对象的等待区等待。在调用栈顶出现,线程状态为WAITING或TIMED_WATING。

**parking to wait for**

park是基本的线程阻塞原语,不通过监视器在对象上阻塞。随concurrent包会出现的新的机制, 和synchronized体系不同。

### 线程动作

线程状态产生的原因

> runnable:状态一般为RUNNABLE。
>
> in Object.wait():等待区等待,状态为WAITING或TIMED_WAITING。
>
> waiting for monitor entry:进入区等待,状态为BLOCKED。
>
> waiting on condition:等待区等待、被park。
>
> sleeping:休眠的线程,调用了Thread.sleep()。

**Wait on condition** 该状态出现在线程等待某个条件的发生。具体是什么原因，可以结合 stacktrace来分析。 最常见的情况就是线程处于sleep状态，等待被唤醒。 常见的情况还有等待网络IO



## 线程Dump的分析

### 原则

> 结合代码阅读的推理。需要线程Dump和源码的相互推导和印证。
>
> 造成Bug的根源往往丌会在调用栈上直接体现,一定格外注意线程当前调用之前的所有调用。

### 入手点

**1 wait on monitor entry进入区等待**：一定是代码有问题

```
"d&a-3588" daemon waiting for monitor entry [0x000000006e5d5000]
java.lang.Thread.State: BLOCKED (on object monitor)
at com.jiuqi.dna.bap.authority.service.UserService$LoginHandler.handle()
- waiting to lock <0x0000000602f38e90> (a java.lang.Object)
at com.jiuqi.dna.bap.authority.service.UserService$LoginHandler.handle()
```

线程状态BLOCKED,线程动作wait on monitor entry,调用修饰waiting to lock总是一起出现。表示在代码级别已经存在冲突的调用。必然有问题代码,需要尽可能减少其发生。

**2 Runnable with IO 同步块阻塞**：

一个线程锁住某对象,大量其他线程在该对象上等待。

```
"blocker-thead-1" runnable
java.lang.Thread.State: RUNNABLE
at com.jiuqi.hcl.javadump.Blocker$1.run(Blocker.java:23)
- locked <0x00000000eb8eff68> (a java.lang.Object)

"blocker-thead-2" waiting for monitor entry
java.lang.Thread.State: BLOCKED (on object monitor)
at com.jiuqi.hcl.javadump.Blocker$2.run(Blocker.java:41)
- waiting to lock <0x00000000eb8eff68> (a java.lang.Object)

"blocker-thead-3" waiting for monitor entry
java.lang.Thread.State: BLOCKED (on object monitor)
at com.jiuqi.hcl.javadump.Blocker$2.run(Blocker.java:41)
- waiting to lock <0x00000000eb8eff68> (a java.lang.Object)
```

 IO操作是可以以RUNNABLE状态达成阻塞。例如:数据库死锁、网络读写。 

>  格外注意对IO线程的真实状态的分析。 一般来说,被捕捉到RUNNABLE的IO调用,都是有问题的。



**持续Runnable的IO是有问题的，常常表示有deadlock。**举例再次说明，以下堆栈显示： 线程状态为RUNNABLE。 调用栈在SocketInputStream或SocketImpl上,socketRead0等方法。 调用栈包含了jdbc相关的包。很可能发生了数据库死锁。更多[数据库常见死锁原因及处理](https://blog.csdn.net/qq_16681169/article/details/74784193)：

```
"d&a-614" daemon prio=6 tid=0x0000000022f1f000 nid=0x37c8 runnable
[0x0000000027cbd000]
java.lang.Thread.State: RUNNABLE
at java.net.SocketInputStream.socketRead0(Native Method)
at java.net.SocketInputStream.read(Unknown Source)
at oracle.net.ns.Packet.receive(Packet.java:240)
at oracle.net.ns.DataPacket.receive(DataPacket.java:92)
at oracle.net.ns.NetInputStream.getNextPacket(NetInputStream.java:172)
at oracle.net.ns.NetInputStream.read(NetInputStream.java:117)
at oracle.jdbc.driver.T4CMAREngine.unmarshalUB1(T4CMAREngine.java:1034)
at oracle.jdbc.driver.T4C8Oall.receive(T4C8Oall.java:588)
```



**3Object.wait() 但是不是线程调度的休眠**

正常的线程池等待

```
"d&a-131" in Object.wait()
java.lang.Thread.State: TIMED_WAITING (on object monitor)
at java.lang.Object.wait(Native Method)
at com.jiuqi.dna.core.impl.WorkingManager.getWorkToDo(WorkingManager.java:322)
- locked <0x0000000313f656f8> (a com.jiuqi.dna.core.impl.WorkingThread)
at com.jiuqi.dna.core.impl.WorkingThread.run(WorkingThread.java:40)
```

可疑的线程等待：下面这个日志是可疑的，上面那个日志是正常的。为啥是可以的，因为它是非线程池等待？

```
"d&a-121" in Object.wait()
java.lang.Thread.State: WAITING (on object monitor)
at java.lang.Object.wait(Native Method)
at java.lang.Object.wait(Object.java:485)
at com.jiuqi.dna.core.impl.AcquirableAccessor.exclusive()
- locked <0x00000003011678d8> (a com.jiuqi.dna.core.impl.CacheGroup)
at com.jiuqi.dna.core.impl.Transaction.lock()
```

#### 4 最后分析一个：jstack信息

```
线程堆栈信息如下：

"Reference Handler" daemon prio=10 tid=0x00007fbbcc06e000 nid=0x286c in Object.wait() [0x00007fbbc8dfc000]
   java.lang.Thread.State: WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    - waiting on <0x0000000783e066e0> (a java.lang.ref.Reference$Lock)
    at java.lang.Object.wait(Object.java:503)
    at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)
    - locked <0x0000000783e066e0> (a java.lang.ref.Reference$Lock)
```

我们能看到：

> 线程的状态： WAITING 线程的调用栈 线程的当前锁住的资源： <0x0000000783e066e0> 线程当前等待的资源：<0x0000000783e066e0>

为什么同时锁住的等待同一个资源：

> 线程的执行中，先获得了这个对象的 Monitor（对应于 locked <0x0000000783e066e0>）。当执行到 obj.wait(), 线程即放弃了 Monitor的所有权，进入 “wait set”队列（对应于 waiting on <0x0000000783e066e0> ）。

### 入手点总结

如果查看现场问题，需要记住3点。总结成话语就是：如果是

- **wait on monitor entry：** 被阻塞的,肯定有问题。（这个没有例子）

- **runnable** ： 注意有没有IO线程，常见于数据库死锁、外部依赖服务失效

- **in Object.wait()**： 注意非线程池等待。

> 项目经验：利用jstack排查CPU占用率100%的线上问题和排查流式应用中的似乎卡住了。

### 1 项目经验：Java程序分析经验谈

如果怀疑一个程序运行不正常，需要从以下几个方面入手？

第一步：看cpu、内存占用。`top -p $PID`

- 若怀疑是CPU问题，`jstack -l $PID`查看卡在什么地方
- 若怀疑GC问题，`jstat -gcutil $PID 1000` 看看GC是否频繁，耗时间情况怎么样。也可以从GC日志获得更准确的信息。调优参数。
- 若怀疑是IO导致的问题, `iostat` 磁盘实时读写速率, `iftop` 查看网络IO的情况
- 若怀疑是上下文切换导致的问题，查看虚拟内存情况：`vmstat` 看整个系统，用` pidstat -lw -p $PID` 看某一个进程的上下文切换。
- 查看进程的网络连接,`netstat -nalp | grep $PID`

第二步: 针对性的看日志

> netstat参数含义：
>
> -a (all)显示所有选项，默认不显示LISTEN相关
>
> -t (tcp)仅显示tcp相关选项
>
> -u (udp)仅显示udp相关选项
>
> -n 拒绝显示别名，能显示数字的全部转化成数字。
>
> -l 仅列出有在 Listen (监听) 的服務状态
>
> -p 显示建立相关链接的程序名
>
> -r 显示路由信息，路由表
>
> -e 显示扩展信息，例如uid等
>
> -s 按各个协议进行统计
>
> -c 每隔一个固定时间，执行该netstat命令。

#### 1.1.一个OLS应用问题：用户觉得程序似乎不工作了，上游数据消费不及时，甚至不消费了。

第一步：看CPU、内存暂用。 

* 怀疑是CPU线程的问题：CPU 0% Mem 0.5%，利用jstack查看，发现符合**入手点总结** 第2条。

* jstack几次无状态变化，卡在Socket读取上。进而排查日志，发现了对应的异常；如果没有日志，应该检查该进程的网络连接。

```
"Apu work thread" prio=10 tid=0x00007f6ad971f000 nid=0x20ade runnable [0x00007f6abd15a000]
   java.lang.Thread.State: RUNNABLE
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.read(SocketInputStream.java:152)
        at java.net.SocketInputStream.read(SocketInputStream.java:122)
        at java.io.BufferedInputStream.fill(BufferedInputStream.java:235)
        at java.io.BufferedInputStream.read1(BufferedInputStream.java:275)
        at java.io.BufferedInputStream.read(BufferedInputStream.java:334)
        - locked <0x0000000760409430> (a java.io.BufferedInputStream)
        at sun.net.www.http.HttpClient.parseHTTPHeader(HttpClient.java:687)
        at sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:633)
        at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1323)
        - locked <0x0000000760409350> (a sun.net.www.protocol.http.HttpURLConnection)
        at postdata.PostDataSpout.readContentFromPost(PostDataSpout.java:126)
        at postdata.PostDataSpout.process(PostDataSpout.java:64)
        at com.sina.ols.apu.process.AbstractProcessUnit$Worker.run(AbstractProcessUnit.java:474)
        at java.lang.Thread.run(Thread.java:745)

   Locked ownable synchronizers:
        - None
```

日志错误

```
java.io.IOException: Server returned HTTP response code: 502 for URL: http://i.app.ent.sina.com.cn/star_active/add/?test=1
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1626)
	at postdata.PostDataSpout.readContentFromPost(PostDataSpout.java:126)
	at postdata.PostDataSpout.process(PostDataSpout.java:64)
	at com.sina.ols.apu.process.AbstractProcessUnit$Worker.run(AbstractProcessUnit.java:474)
	at java.lang.Thread.run(Thread.java:745)
java.net.ConnectException: Connection refused
	at java.net.PlainSocketImpl.socketConnect(Native Method)
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:339)
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:200)
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:182)
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
	at java.net.Socket.connect(Socket.java:579)
	at java.net.Socket.connect(Socket.java:528)
	at sun.net.NetworkClient.doConnect(NetworkClient.java:180)
	at sun.net.www.http.HttpClient.openServer(HttpClient.java:432)
	at sun.net.www.http.HttpClient.openServer(HttpClient.java:527)
	at sun.net.www.http.HttpClient.<init>(HttpClient.java:211)
	at sun.net.www.http.HttpClient.New(HttpClient.java:308)
	at sun.net.www.http.HttpClient.New(HttpClient.java:326)
	at sun.net.www.protocol.http.HttpURLConnection.getNewHttpClient(HttpURLConnection.java:996)
	at sun.net.www.protocol.http.HttpURLConnection.plainConnect(HttpURLConnection.java:932)
	at sun.net.www.protocol.http.HttpURLConnection.connect(HttpURLConnection.java:850)
	at postdata.PostDataSpout.readContentFromPost(PostDataSpout.java:112)
	at postdata.PostDataSpout.process(PostDataSpout.java:64)
	at com.sina.ols.apu.process.AbstractProcessUnit$Worker.run(AbstractProcessUnit.java:474)
	at java.lang.Thread.run(Thread.java:745)
```

#### 1.2 上下文切换导致服务响应慢

解决多线程上下文切换太多，导致test-service响应慢的问题： 实际问题是：用户需要查询ES的数据，偶尔会很慢。这个问题，不是每次都能复现。后来，我决定用ab压测这个接口，发现发现上下文切换cs很高，达到几万，耗时很长，达到秒级。原因是这个地方，开启了太多的100个线程去并发查询，每次只查询5个，导致了上下文切换很高。这个地方，没有必要使用多线程设计，我采用少量的线程比如6查询，10ms级就返回了，cs自然就降低到几十了。所以对于快速的操作，不适合开多线程来进行。（什么操作会导致上下文频繁切换？）

回答2个问题：

- 上下文切换是什么
  - 一个线程被另外一个线程暂停，并占用了CPU。
- 上下文切换代价高在哪里？
  - 操作系统需要保存上下文
  - 线程要重新调度
  - CPU缓存会被重新加载
- 什么时候单线程，什么时候多线程，给的启示是什么呢？
  - 业务简单，快速，及时ms，用单线程+队列，比如查询Redis缓存数据，ES估计也可以。
  - 有IO瓶颈 或者耗时太长，就用多线程，比如NIO读写