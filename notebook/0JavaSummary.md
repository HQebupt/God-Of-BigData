## 0 jdk版本重要特性

- jdk7
  - 增加fork/join并发架构
  - 基本整型可表达为二进制方式
  - switch语句支持字符串
  - 新增try-with-resources语句
  - 多重try-catch语句、精确的rethrow类型匹配
  - 数值字面量可用下划线分割以增加可读性
  - 泛型初始化自动类型推断(new HashMap不需要在尖括号中写类型了)
  - JVM引入G1收集器
  - 分层编译优化提升客户端VM启动速度
- jdk8
  - lambda表达式、新增java.util.stream
  - 提升HashMap碰撞时的查询效率
  - 增强自动类型推断
  - 并行Array排序
  - Java Flight Recorder
- jdk9
  - 将G1提升为默认垃圾收集器，修复问题、提升性能，实现默认基本配置运行而不需要手动配置
  - 删除一些收集器配置，如-XX:+CMSIncrementalMode、-XX:+UseCMSCompactAtFullCollection、-XX:+CMSFullGCsBeforeCompaction、-XX:+UseCMSCollectionPassing，-XX:+UseParNewGC不再使用，只能默认与CMS配对使用
- jdk10
  - G1GC的FullGC并行化
  - 本地变量类型推断
- jdk11
  - String新增方法：isBlank、lines、strip、stripLeading、stripTailing、repeat
  - var类型推断增强
  - FlightRecorder开源免费
  - 引入ZGC（expiremental，只在linux/x64上可用）
- jdk12
  - ZGC并发卸载类

## 1常见的Java问题

- ArrayList, LinkedList, HashSet, HashMap, CopyOnWriteList, ConcurrentHashMap, ConcurrentLinkedQueue, volatile, Atomic, CAS, double check locking, happen-before, Object Header, false-sharing, ThreadExecutor(Cached, Fixed, Scheduled), syncronize, Lock, CountdownLatch, Barrier, Exchanger, JVM survivor, GC Algorithem, JVM tuning, syncronize tuning in JDK1.6, strong/weak/phantom reference, String pool

### HashMap

- 两个参数影响其性能：initial capacity、load factor
  - 当哈希表的 Entry 个数达到二者的乘积，就会触发 rehash
  - load factor 默认为0.75是一个在时间和空间消耗上较好的折中
  - 过高的 load factor 值降低空间开销，但是会增加查找的时间消耗
- 非线程安全，可通过外部线程同步，或者初始化时调用

```
    Map m = Collections.synchronizedMap(new HashMap(...))
```

- capacity都是2的次幂
  - hash 值算法是 hashCode ^ (hashcode>>>16) 后取模运算，通过 h & (length-1) 取模
    - 这样使得一个 hash 值的有效位只有 length-1 ，因此需要高 16 位与低 16 位 XOR 扩散(TODO)
  - key 碰撞时先存成链表，链表长度 > 8 则重构为红黑树
  - 1.8 扩容的时候 rehash 更优雅
    - 每次扩容都 *2，因此 resize 时，不需要重新计算 hash，只需要看 hash 值新增的 bit 是 1 还是 0，是 0 则索引不变，是 1 则 索引 = 原索引 + oldCap。
- 老版本并发扩容时可能导致形成环形链表导致读取死循环

- getNode代码
  - 用key的hash值去查询内部数组tab能否命中；如果没命中，就返回null。
  - 命中之后，再次比较key是不是相等。根据key是红黑树还是链表来查找

```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

- putVal代码

  - 如果内部数组Table为空，先初始化表（resize）
  - 计算 key的hash 值，在数组的对应 index 上插入新值，如果当前数组对应位置为空，直接插入
  - 如果数组上对应位置已有值
    - 判断该 Node 是否为 TreeNode，如果是 TreeNode 则直接插入
    - 如果是 List，在链表尾部插入的同时，记录 binCount，若 binCount > 8 将链表转换为树

  ![image-20210719222037297](0JavaSummary.assets/image-20210719222037297.png)
  
  ```java
      final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                     boolean evict) {
          Node<K,V>[] tab; Node<K,V> p; int n, i;
          if ((tab = table) == null || (n = tab.length) == 0)
              n = (tab = resize()).length;
          if ((p = tab[i = (n - 1) & hash]) == null)
              tab[i] = newNode(hash, key, value, null);
          else {
              Node<K,V> e; K k;
              if (p.hash == hash &&
                  ((k = p.key) == key || (key != null && key.equals(k))))
                  e = p;
              else if (p instanceof TreeNode)
                  e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
              else {
                  for (int binCount = 0; ; ++binCount) {
                      if ((e = p.next) == null) {
                          p.next = newNode(hash, key, value, null);
                          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                              treeifyBin(tab, hash);
                          break;
                      }
                      if (e.hash == hash &&
                          ((k = e.key) == key || (key != null && key.equals(k))))
                          break;
                      p = e;
                  }
              }
              if (e != null) { // existing mapping for key
                  V oldValue = e.value;
                  if (!onlyIfAbsent || oldValue == null)
                      e.value = value;
                  afterNodeAccess(e);
                  return oldValue;
              }
          }
          ++modCount;
          if (++size > threshold)
              resize();
          afterNodeInsertion(evict);
          return null;
      }
  
  ```

### ConcurrentHashMap

HashTable 各个操作都是加锁的，ConcurrentHashMap的get 不加锁，put加锁。这会造成get的时候不一定拿到最新的数据，属于弱一致性的实现。因此，强一致性的场景，用HashTable。

@jdk7 使用分段锁设计；iterator 不会抛 concurrentModification 异常（造成读会读到老的数据），但是只能被一个线程操作；size 方法开销很大 。@jdk1.8之后，使用Synchronized同步锁。

<img src="0JavaSummary.assets/120290-1622708615286" alt="img" style="zoom: 67%;" />

```java
   public V put(K key, V value) {
            Segment<K,V> s;

            if (value == null)
                throw new NullPointerException();

            int hash = hash(key);

            // segmentMask：段掩码，假如segments数组长度为16，则段掩码为16-1=15；segments长度为32，段掩码为32-1=31
            // segmentShift：2的sshift次方等于ssize，segmentShift=32-sshift
            // 若segments长度为16，segmentShift=32-4=28
            // 若segments长度为32，segmentShift=32-5=27, 而计算得出的hash值最大为32位，无符号右移segmentShift，
            // 则意味着只保留高几位（其余位是没用的），然后与段掩码segmentMask位运算来定位Segment
            int j = (hash >>> segmentShift) & segmentMask;
            if ((s = (Segment<K,V>) UNSAFE.getObject          // nonvolatile; recheck
                 (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
                s = ensureSegment(j);
            return s.put(key, hash, value, false);
	}
```



@jdk8 改为 CAS 设计

- 对于一个空的Node，CAS无锁添加；
- 对于非空的Node，对Node加锁（synchronized，锁可以被优化）；
- 增加 addCount 方法专门记录 size ；并发修改一个下标的 Node 时才加 synchronized ，并且只锁定当前的 Node



- isEmpty、size、containsValue 只在 map 不并发更新的时候准确，适合用于监控或估算，而不适于程序控制；不支持 Null Key/Value

- putVal

  - 在没有哈希冲突的情况下，使用CAS添加；有冲突，就用Synchronized把链表Node锁住，还是锁住的一个链表，不是整个map。
  - 计算 hash 值，自旋访问 table，table 为空则采用 CAS 初始化 table
  - 获取 hash 值对应节点位置 i，若该位置为空则 CAS 插入
  - 若有HashMap在扩容，则先执行 helpTransfer 帮助迁移到新 table。会再次自旋进入循环体。
  - 否则，以当前节点的链表、树的头结点为 lock 加锁，进行 add 操作
  
  > 可以发现，和HashMap的putVal流程很类似，区别在于由于并发性，ConcurrentHashMap有CAS操作和synchronized。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode()); // 1 计算hash值
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) { // 1 自旋访问table
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); // 1 为空CAS初始化table
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 2 获取table中对应Node i位置，如果位置为空，直接CAS插入
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED) // 3 如果HashMap在扩容，执行helpTransfer迁移到新table
                tab = helpTransfer(tab, f);
            else { // 4 否则，说明一切正常，是链表Node就添加，是TreeNode也添加。因为是并发的map，所以这个地方加锁
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

- get操作：比较简单，就是用key的hash值去命中Node。没有加锁，使用volatile (修饰table的) 来保证可见性。这就是所谓的弱一致性。

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) { // 1 用key的hash值去命中Node
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0) // 2 遇到resize，去寻找新的地方寻找key
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) { // 3 链表式查找
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

#### size优化

- 1.8中的 size 增加 baseCount、counterCells 来辅助记录 size，优化性能

```java
	/**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount;

    private transient volatile CounterCell[] counterCells;

    /**
     * A padded cell for distributing counts.  Adapted from LongAdder
     * and Striped64.  See their internal docs for explanation.
     */
    @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }
```

- put方法结束后，调用 addCount 方法以 CAS 的方式自增 baseCount，如果 CAS 失败则使用 CAS 记录到 counterCell 中

```java
	private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
	......
```

* 如果记录 counterCell 的 CAS 失败则调用 fullAddCount 继续自旋 CAS 直到成功

#### 适用场景

ConcurrentHashMap: 数组+链表+红黑树+锁。红黑树在并发的情况下，删除和插入过程中，需要平衡，会操作大量的节点，因此竞争锁资源激烈，代价相对于跳表高。

> 因此，在单线程Map容器中，TreeMap容易来存取大数据量；线程安全的case下，SkipListMap来存大数据。

![image-20210719222931125](0JavaSummary.assets/image-20210719222931125.png)

1. HashMap 底层设计与实现，LoadFactor，为什么多线程环境下，put方法会造成死循环？为什么size>=8 之后，要转换成红黑树？红黑树能手写一个么？
2. HashTable \ ConcurrectHashMap分别对应什么样的适用场景？
3. ConcurrentHashMap 1.8之前是segment lock, 之后使用synchronized和CAS来更新，为什么？涉及1.6之后，对synchronized的优化，重量级锁、偏向锁、自旋锁
4. ConcurrentSkipListMap 基于跳表实现的Map，优点是什么，对比ConcurrentHashMap的适用场景是什么？
5. 互联网电商系统，往往都有黑名单，你觉得用什么样的数据结构来存储？如果要设计一个统计商品销量Top10的功能，用什么样的数据结构？
6. 如果使用队列来实现抢购的排队，如何选择队列？特点是写多读少。

### 跳表

优化搜索的有序链表

### CopyOnWriteArrayList

特点：读，无所并发；写：复制在更新，copy回去。（读写分离的并发思想）

适用于什么场景？

### 多线程

- ThreadLocalRandom 、ThreadLocal，解决多线程访问同一个变量的时候需要同步，让每个线程都拷贝一份变量，自己操作自己。
- 线程池的工作原理、任务调度

<img src="0JavaSummary.assets/31bad766983e212431077ca8da92762050214.png" alt="图4 任务调度流程" style="zoom:80%;" />

- 线程池ThreadPoolExecutor，关键的属性

  - corePoolSize，一旦有新任务提交，而线程池当前线程数小于此值，则 create 新的线程（即 core thread 只在新任务提交时创建，但是可以通过调用 prestartCoreThread 、prestartAllCoreThreads 方法改变策略）

  - maximumPoolSize，一旦有新任务提交，而线程池当前线程数小于此值且大于 corePoolSize，则进入队列排队，只有队列被填满才会创建新线程

  - Keep-alive times，超过 corePoolSize 的线程如果空闲时间超过此值，就会被终止

  - Queueing，若小于 corePoolSize 的线程在运行，则 executor 更倾向于增加新线程而不是排队；反之则倾向于排队；若队列满，则创建新线程；若队列满且线程数超过 maximumPoolSize 则 reject；三种排队策略：

    - Direct handoffs，SynchronousQueue，不持有任务，接收到任务直接转交给Executor（newCachedThreadPool采取的是此策略）
    - Unbounded queues，LinkedBlockingQueue，若任务提交数超过 corePoolSize 支持的线程，新任务会持续添加进队列，而 maximumPoolSize 不会生效（newSingledThreadPool、newFixedThreadPool采取的是此策略）
    - Bounded queues，ArrayBlockingQueue，Queue sizes 和 maximum pool 需要相互权衡

  - Reject任务后的策略：

    - AbortPolicy，抛异常 RejectedExecutionException（默认策略）
    - CallerRunsPolicy，交由提交线程自己去执行 execute
    - DiscardPolicy，丢弃任务
    - DiscardOldestPolicy，丢弃队列头部任务
    - 自定义 RejectedExecutionHandler

  - Hook Methods，beforeExecute，afterExecute，若钩子函数或 callback 调用异常，线程会被终止

  - Executors常见的 ThreadPool 参数

    - cachedThreadPool

    ```java
        new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    ```

    - fixedThreadPool

    ```java
    	new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
    ```

    

## 2 Java锁

![img](0JavaSummary.assets/7f749fc8.png)

### 1. 乐观锁 VS 悲观锁

- 悲观锁：取数据的时候会先加锁。（synchronized和lock实现）

  - 适用场景：写操作多。

- 乐观锁：不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。

  - 若没有，就更新。
  - 若有，则取决于具体实现

  * 最常实现是CAS算法，Java原子类中的递增操作就通过CAS自旋实现的。
  * 适用场景：读操作多。

#### CAS

CAS算法，三个操作数：

- 需要读写的内存值 V。
- 更新前，获取内存值 A。
- 要写入的新值 B。（只有当V==A的时候，才会进行更新）

```java
// ------------------------- JDK 8 -------------------------
// AtomicInteger 
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe(); // 操作内存的类
private static final long valueOffset; // 存储value在AtomicInteger中的偏移量。
private volatile int value; // 存储实际的值，需要借助volatile关键字保证其在线程间是可见的。

public final int incrementAndGet() {
  return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// ------------------------- OpenJDK 8 -------------------------
// Unsafe.java
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta)); // 底层实现是CPU指令CMPXGHG，原子操作。
   return v;
}
```

- ABA 问题的解决：添加版本号，变成1A－2B－3A（AtomicStampedReference）
- 循环时间长开销大
- 只能保证一个共享变量的原子操作。(AtomicReference)

### 2. 自旋锁 VS 适应性自旋锁

阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，耗费处理器时间。

- 自旋锁：某线程尝试获取同步资源失败后，不放弃CPU时间片，通过自旋等待锁释放。（CAS是原理）
  - 自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

- 自适应自旋锁：自旋的次数不在固定，由虚拟机来决定。

  - 如果一个锁，自旋经常成功，那么虚拟机就回给它留多的次数来自旋
  - 如果一个锁，自旋经常失败，那么虚拟机可能就直接让获取锁变成阻塞，不再采用自旋的方式。

  - JDK 6中变为默认开启，并且引入了自适应的自旋锁（适应性自旋锁）。

### 3. 无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁

synchronized锁的状态

- synchronized，悲观锁，这把锁就是存在Java对象头里的

  - Mark Word（标记字段）、Klass Pointer（类型指针）。
  - Mark Word：存储HashCode，分代年龄和锁标志位信息。

#### Monitor

- 同步机制。每一个Java对象都有

- monitor record列表，线程私有的数据结构
- monitor的Owner字段存放拥有该锁的线程的唯一标识

<img src="0JavaSummary.assets/image-20210610101721136-1623291442906.png" alt="image-20210610101721136" style="zoom: 50%;" />



**synchronized通过Monitor来实现线程同步，Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的线程同步。**

> Mutex Lock实现：CPU指令，swap或exchange指令，是把寄存器和内存单元的数据相交换。
>
> ```python
> lock: 
> 	if(mutex > 0){ # mutex = 1 表示锁空闲
> 		mutex = 0;   # mutex = 0 表示锁占用
> 		return 0; 
> 	} else 
> 		挂起等待; 
> 	goto lock;
> 
> unlock: 
> 	mutex = 1; 
> 	唤醒等待Mutex的线程; 
> ```
>
> 

因为它依赖于操作系统的互斥锁来实现的。我们称之为“重量级锁“

- 四种锁状态对应的Object对象头

| 锁状态   | 存储内容                                                | 存储内容 |
| :------- | :------------------------------------------------------ | :------- |
| 无锁     | 对象的hashCode、对象分代年龄、是否是偏向锁（0）         | 01       |
| 偏向锁   | 偏向线程ID、偏向时间戳、对象分代年龄、是否是偏向锁（1） | 01       |
| 轻量级锁 | 指向栈中锁记录的指针                                    | 00       |
| 重量级锁 | 指向互斥量（重量级锁）的指针                            | 10       |

#### 3.1 无锁

- CAS是无锁的实现

#### 3.2**偏向锁**

**偏向锁是指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。**

- 背景：引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令即可。
- 适用场景：在只有一个线程执行同步代码块时能够提高性能。
- 怎么做到的：当一个线程访问同步代码块并获取锁时，会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁。
- 什么时候释放：只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动释放偏向锁。（撤销，需要等待全局安全点）
- 默认开启，如果需要关闭偏向锁：-XX:-UseBiasedLocking=false，关闭之后程序默认会进入轻量级锁状态。

#### 3.3**轻量级锁**

**是指当锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。**

#### 3.4 重量级锁

若当锁是轻量级锁的时候，当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁。

#### Synchronized优化

- 无锁：当对象被synchronized修饰，先处于无锁状态，Mark Word没有存储锁信息（无锁没有对资源进行锁定，所有的线程都能访问并修改同一个资源，但同时只有一个线程能修改成功。CAS实现）
- 偏向锁：当对象被同一个线程访问，把它的Thread ID储存在Mark Word里，进入偏向锁。（这个状态下，线程访问这个对象的时候，不在通过CAS来加锁，直接检测Mark Word的ThreadID值）
- 轻量级锁：偏向锁时，如果有第二个线程来了，那么锁就升级成轻量级锁。它就会自旋等待第一个线程释放锁。（第一个线程会在全局安全点的时候，才会释放锁）
- 重量级锁：轻量级锁时，自旋等待时间太长了，或者又有第3个线程来竞争，那么锁升级，阻塞其它线程。

  - 偏向锁通过对比Mark Word解决加锁问题，避免执行CAS操作。
  - 轻量级锁是通过用CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒。
  - 重量级锁是将除了拥有锁的线程以外的线程都阻塞。

### 4. 公平锁 VS 非公平锁

- 公平锁：竞争资源是否需要排队
- 非公平锁：先尝试插队，失败再排队。

通过ReentrantLock的源码来理解公平锁和非公平锁。

<img src="0JavaSummary.assets/6edea205-1622776296892.png" alt="img" style="zoom: 50%;" />

**ReentrantLock里面有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer）**

公平锁与非公平锁的加锁方法的源码:

<img src="0JavaSummary.assets/bc6fe583.png" alt="img" style="zoom:50%;" />

- hasQueuedPredecessors()：主要是判断当前线程是否位于同步队列中的第一个

#### 4.1 **AbstractQueuedSynchronizer**

- **基于原子变量 state 和 queue 实现的同步框架，state == 1 表示锁已被抢占，state ==0 表示空闲**

```java
    /**
     * The synchronization state.
     */
    private volatile int state;
```

<img src="0JavaSummary.assets/image-20210604145049525-1622789451026.png" alt="image-20210604145049525" style="zoom:67%;" />

- 核心方法
  - tryAquire()、tryRelease()
  - tryAcquireShared()、tryReleaseShared()
  - isHeldExclusively()
  - getState()、setState()、compareAndSetState()
- 核心成员变量
  - state
  - head
  - tail
- Node 成员变量

```
    int waitStatus;
    Node prev;
    Node next;
    Thread thread;
    Node nextWaiter;
```

- waitStatus 用于标识等待队列中 Node 的状态（这样的设计不仅在于用于标识需要挂起的线程，同时也避免了出现异常而退出或者被取消的线程占据队列空间导致其他等待线程饥饿）
  - SIGNAL = -1 // 表示后继节点当前被阻塞（或即将被阻塞），需前驱节点释放时解除这个后继几点的阻塞
  - CANCELLED = 1
  - CONDIGION = -2
  - PROPAGATE = -3 // 用于公平锁

```
    volatile int waitStatus;
```

- **AQS 父类(AbstractOwnableSynchronizer)用于记录当前获得锁的线程**
  - setExclusiveOwnerThread()
  - getExclusiveOwnerThread()
- 虽然 AQS 基于内部内部 FIFO 队列, 但是它获取锁的策略不一定是FIFO的，一个排他锁的核心形式如下: 先尝试获取锁，不成功再入队。

```java
    Acquire:
       while (!tryAcquire(arg)) {
          enqueue thread if it is not already queued;
          possibly block current thread;
       }

    Release:
       if (tryRelease(arg))
          unblock the first queued thread;
```

#### 4.2 ReentrantLock 原理

- 基于 AQS ，初次竞争使用 CAS 将 status 置为 1，若成功则抢到锁，再将独占线程置为自身（因此在竞争不频繁时效率很高）

```java
 public void lock() {
    	sync.lock();
    }

    static final class NonfairSync extends Sync {
    	...

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
            }
        }
        ...
    }
```

- 以上 CAS 失败则调用 AQS.acquire()，在分支中又会调用 tryAcquire，
  - 如果调用的是 NonFairAcquire
    - tryAcquire 判断当前 status
    - 若为 0 则尝试用 CAS 置 status 为 1
    - CAS 失败则判断是否为当前线程重入该锁，是则获得锁，否则加入等待队列
  - 如果调用的是 FairAcquire
    - 逻辑与 NonFairAcquire 一致，只是在抢锁前线判断等待队列是否为空

```java
// in AbstractQueuedSynchronizer
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    // in NotfairSync
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

    // in Sync
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { // 可重入判断
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    // FairSync
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() && // 检查队列是否有等待线程
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

- 如果tryAcquire失败了，就会把线程入队。若队列未初始化，则初始化头结点（thread=null, waitstatus=0），然后添加到队列尾；然后开始进入 acquireQueued 中的 循环 ①

```java
	private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node); // 入队
        return node;
    }

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
            }
        }
    }

```

- **进入等待队列之后，还会再次尝试获取锁。**若自身为首节点（head后的第一个节点），则尝试获得锁 tryAcquire，若能获得，则弹出头结点，返回；否则判断当前线程是否应当挂起，如果节点刚被初始化，则前置节点的 waitstatus 为 0，则将 waitstatus 改为 SIGNAL 后，再次进入 外循环 ① 尝试抢锁，失败则再次进入判断是否应当挂起的逻辑，正常情况下，第二次循环如果还没得到锁，就会被挂起

```java
   final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) { // 循环①
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node); // node的thread和prev都被置空, head = node
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) && // 没有抢到锁就标记自身应当被挂起，等待下次循环再挂起
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }

    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

```

- 下面是release的过程：当获得锁的线程 release 时，会将首节点唤醒，唤醒的首节点会再次进入 循环 ①，执行之前的自旋抢锁的逻辑，此时该节点的线程会和其他新来而未进队列的线程一起竞争锁，因此它并不一定能抢得到，从这个角度来看，它并不比新来的线程更优先，但是比队列中的其他线程都更优先

```java
	// in AbstractQueuedSynchonizer
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

	// in ReentrantLock#Sync
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```

###  5 可重入锁 VS 非可重入锁

可重入锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class）

Java中ReentrantLock和synchronized都是可重入锁，一定程度避免死锁。下面用示例代码来进行分析：后面有一张图是ReentrantLock的代码Sync的一部分。

```java
public class Widget {
    public synchronized void doSomething() {
        System.out.println("方法1执行...");
        doOthers();
    }

    public synchronized void doOthers() {
        System.out.println("方法2执行...");
    }
}
```

> ReentrantLock的可重入是通过Sync来实现的，Sync是AQS实现的，获取锁tryAccquire的时候。
>
> Synchronized的可重入是怎么实现的？



![img](0JavaSummary.assets/32536e7a.png)

### 6 独享锁 VS 共享锁

独享锁和共享锁同样是一种概念。

- 独享锁也叫排他锁、互斥锁，是指该锁一次只能被一个线程所持有。如果线程T对数据A加上排它锁后，则其他线程不能再对A加任何类型的锁。获得排它锁的线程即能读数据又能修改数据。**JDK中的synchronized和JUC中Lock的实现类就是互斥锁。**

- 共享锁是指该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A再加共享锁，不能加排它锁。获得共享锁的线程只能读数据，不能修改数据。

ReentrantLock和ReentrantReadWriteLock的源码，独享锁和共享锁都是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享。

下图为ReentrantReadWriteLock的部分源码：

![img](0JavaSummary.assets/762a042b-1622794632079.png)

- ReadWriteLock，实现类 ReentrantReadWriteLock

  - 获取顺序：此类不会将读取者优先或写入者优先强加给锁访问的排序
    - 非公平模式（默认）：连续竞争的非公平锁可能无限期地推迟一个或多个 reader 或 writer 线程，但吞吐量通常要高于公平锁
    - 公平模式：线程利用一个近似到达顺序的策略来争夺进入。当释放锁时，可以为等待时间最长的那个 writer 线程分配写入锁，如果有一组 reader 的等待时间大于所有正在等待的 writer 线程，将为该组分配读者锁
    - 试图获得公平写入锁的非重入的线程将会阻塞，除非读取锁和写入锁都已释放（这意味着没有等待线程）
  - 排他性
    - 读读共享，读写互斥，写写互斥
  - 可重入性
    - 允许 reader 和 writer 按照 ReentrantLock 的样式重新获取读取锁或写入锁
    - 在写入线程持有的所有写入锁都已经释放后，才允许重入 reader 使用读取锁
    - **writer 可以获取读取锁，但 reader 不能获取写入锁（也就是说，在正在读的时候，同一个线程可以进入来进行写操作。这和下面这一点是相关的）**
  - **锁降级：重入允许从写入锁降级为读取锁**
    - 先获取写入锁，然后获取读取锁，最后释放写入锁
    - 但是，从读取锁升级到写入锁是不可能的

  ```java
  // 实现一个线程安全的可以查的字典数据    
  class RWDictionary {
         private final Map<String, Data> m = new TreeMap<String, Data>();
         private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
         private final Lock r = rwl.readLock();
         private final Lock w = rwl.writeLock();
  
         public Data get(String key) {
           r.lock();
           try { return m.get(key); }
           finally { r.unlock(); }
         }
         public String[] allKeys() {
           r.lock();
           try { return m.keySet().toArray(); }
           finally { r.unlock(); }
         }
         public Data put(String key, Data value) {
           w.lock();
           try { return m.put(key, value); }
           finally { w.unlock(); }
         }
         public void clear() {
           w.lock();
           try { m.clear(); }
           finally { w.unlock(); }
         }
      }
  ```

那读锁和写锁的具体加锁方式有什么区别呢？在了解源码之前我们需要回顾一下其他知识。 在最开始提及AQS的时候我们也提到了state字段（int类型，32位），该字段用来描述有多少线程获持有锁。

在独享锁中这个值通常是0或者1（如果是重入锁的话state值就是重入的次数），在共享锁中state就是持有锁的数量。但是在ReentrantReadWriteLock中有读、写两把锁，所以需要在一个整型变量state上分别描述读锁和写锁的数量（或者也可以叫状态）。于是将state变量“按位切割”切分成了两个部分，高16位表示读锁状态（读锁个数），低16位表示写锁状态（写锁个数）。如下图所示：

![img](0JavaSummary.assets/8793e00a-1622796205188.png)

了解了概念之后我们再来看代码，先看写锁的加锁源码：

```Java
        final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);// 当前写锁的个数w
                if (w == 0 || current != getExclusiveOwnerThread()) // 读写互斥
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

读锁源码

```java
		final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current) // 可重入判断
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```

> 反向思考：ReentrantLock里面的公平锁和非公平锁获取的时候，`tryAcquire` `nonfairTryAcquire` 他们是独占锁还是共享锁？

Refer:[Java锁事](https://tech.meituan.com/2018/11/15/java-lock.html)

### 7 信号量Semaphore

操作系统的信号量是一种概念，Java的信号量是一种实现。

- **信号量是一个被线程共享的非负变量。是一个发信号的机制。**一个等待一个信号量的线程可以被其他线程通知（signal）。这个机制通过 wait 和 signal 两个原子操作（atomic operations）来实现进程同步。

- 互斥锁，Mutex，Mutual Exclusion Object，**互斥锁是一个互斥对象。它是一种特殊的二进位信号量（binary semaphore）**，用来控制访问共享区域资源。

  


#### 信号量和互斥锁的不同点

| 参数     | 信号量                                                       | 互斥锁                                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 机制     | **是一种发信号的机制（signaling mechanism）**                | **是一种锁机制**                                             |
| 数据类型 | **信号量是一个整型变量**                                     | **斥锁是一个对象**                                           |
| 修改     | 等待（wait）和发信号（signal）操作可以修改信号量             | 互斥锁只有当进程请求访问一块资源或释放占用某块资源的时候被修改 |
| 资源管理 | 如果没有空闲资源，此时请求资源的进程将执行等待操作（wait operation）。它将一直等待直到信号量的计数大于0 | 如果互斥锁是锁住的状态，进程只能等待。进程将被置于队列中进行排队。只有当互斥锁被解锁后才能访问资源 |
| 线程     | 可以拥有多个线程                                             | 可以拥有多个线程，但是多个线程不是同时进行的                 |
| 所有权   | 任一进程释放或者获取资源时，计数将被改变                     | 锁对象只能被当前获取到钥匙的进程释放（即工作线程）           |
| 类型     | 信号量有两种类型，二进位信号量和计数信号量                   | 互斥锁没有子类型                                             |
| 操作     | 信号量的值可以通过等待和发信号两个操作来修改                 | 互斥锁有锁上（locked）和解锁（unlocked）两个操作             |
| 资源占用 | 如果所有资源都被占用，此时请求资源的进程将执行wait（）操作并阻塞自身，直到信号量计数>1，占有被释放的资源 | 如果对象已被锁定，则请求资源的进程将等待，并在释放锁定之前由系统进行排队 |
| 优缺点   | 信号量可以灵活管理资源；缺点是编程复杂，容易出现死锁。       |                                                              |

Refer [雨幻逐光](https://www.jianshu.com/p/5852efef0ea8)

```java
/*A counting semaphore. Conceptually, a semaphore maintains a set of permits. Each acquire blocks if necessary until a permit is available, and then takes it. Each release adds a permit, potentially releasing a blocking acquirer. However, no actual permit objects are used; the Semaphore just keeps a count of the number available and acts accordingly.
Semaphores are often used to restrict the number of threads than can access some (physical or logical) resource. For example, here is a class that uses a semaphore to control access to a pool of items:*/
  
 class Pool {
   private static final int MAX_AVAILABLE = 100;
   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

   public Object getItem() throws InterruptedException {
     available.acquire();
     return getNextAvailableItem();
   }

   public void putItem(Object x) {
     if (markAsUnused(x))
       available.release();
   }

   // Not a particularly efficient data structure; just for demo

   protected Object[] items = ... whatever kinds of items being managed
   protected boolean[] used = new boolean[MAX_AVAILABLE];

   protected synchronized Object getNextAvailableItem() {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (!used[i]) {
          used[i] = true;
          return items[i];
       }
     }
     return null; // not reached
   }

   protected synchronized boolean markAsUnused(Object item) {
     for (int i = 0; i < MAX_AVAILABLE; ++i) {
       if (item == items[i]) {
          if (used[i]) {
            used[i] = false;
            return true;
          } else
            return false;
       }
     }
     return false;
   }
 }
```

- Java 信号量还是AQS实现

```java
public class Semaphore implements java.io.Serializable {
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;

    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }


    /**
     * Creates a Semaphore with the given number of
     * permits and nonfair fairness setting.
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
}
```



### 8 synchronized区别ReentrantLock

- Synchronized，重量级，线程切换，耗费系统资源，非公平锁，不可中断，一个等待队列
- ReentrantLock，轻量级，不切换线程，cas+volatile，选择公平，选择中断，Condition等待队列
  - 适用场景：时间锁、可中断锁、多个条件变量

## 3 JVM

<img src="0JavaSummary.assets/image-20210712082721625.png" alt="image-20210712082721625" style="zoom: 33%;" />



### 四种引用

- 强引用:无论什么时候都不会自动回收（场景：正常使用）
- 软引用:空间不足才会回收对象（场景：适合缓存）
- 弱引用:不是立刻回收而是GC发现后会在下次GC时才会回收对象。（场景：如果对象偶尔使用可WeakReference.）
- 虚引用: 主要用来跟踪对象被垃圾回收的活动。必须和引用队列(ReferenceQueue)联合使用,本次GC发现立即回收对象（场景：GC里面使用？）

StrongReference、WeakReference、SoftReference、PhantomReference

- SoftReference  空间不足才会回收对象（场景：适合缓存）

```java
		String str = new String("abc");		
		SoftReference<String> softReference = new SoftReference<String>(str);		// 浏览器的回退可以用缓存设计：    
		if(softReference.get() != null) {        
      	page = softReference.get(); // 内存充足，还没有被回收器回收，直接获取缓存    
    } else {        
      page = browser.getPage();// 内存不足，软引用的对象已经回收        
      softReference = new SoftReference(page);// 重新构建软引用    
    }
```

- WeakReference 引用的对象，当没有强引用指向它后，将在 GC 时被回收；如果其作为 Map.Entry 中的key，则整个 Entry 会被移除。

```java
    Obejct reference = new Object();    
		WeakReference<Obejct> weakRef = new WeakReference<>(reference);    
		reference = null;    
		System.gc(); // 被回收    
		AssertNull(weakRef.get())
```

- PhantomReference，调用 get 永远返回 null，用于跟踪引用何时被 enqueue 至 ReferenceQueue 中.**虚引用必须和引用队列(ReferenceQueue)联合使用**。

  > 对于GC来看，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，会在回收之前，把虚引用加入到与之关联的引用队列中。
  >
  > 对于用户来看，如果程序发现某个虚引用已经被加入到ReferenceQueue，那么回收之前采取一些行动。

```java
String str = new String("abc");  
ReferenceQueue queue = new ReferenceQueue();    // 创建虚引用，要求必须与一个引用队列关联    
PhantomReference pr = new PhantomReference(str, queue);
```



### 结构、回收算法

- 基本结构：程序计数器、JVM 栈、native 栈、堆、运行时常量池
- 分区：垃圾收集器把堆分为新生代、老年代、永久代，1.8 版本后引入了 MetaSpace 替换永久代，本地内存。
- 直接内存: NIO，DirectByteBuffer分配
- 基本回收算法：标记清除，标记整理，复制
- 对象分配策略
  - 优先分配在新生代
  - 大对象直接老年代(-XX:PretenureSizeThreshold)
  - 长时间存活对象进入老年代(- XX:MaxTenuringThreshold)
  - 动态年龄判定：若 Survivor 中相同年龄的对象大小和 > Survivor 空间的一半，则年龄大于等于该值的对象直接进入老年代
  - 空间分配担保策略
    - YGC 前 JVM 检查老年代的连续空间是否大于新生代所有对象总和
    - 若上述为否且 HandlePromotionFailure 为 true ，则检查老年代连续空间是否大于每次晋升对象的平均大小
    - 若上述为否，或 HandlePromotionFailure 为 false ，则触发 FullGC
- GC Roots和对象路由
  - GC Roots 包括：活动线程相关的各种引用、静态变量、JNI 引用等
  - 垃圾收集器内有一组成为 OopMap 的数据结构存储了所有对象的地址
  - <img src="0JavaSummary.assets/Cgq2xl4hefWAWKFZAAMwndGjScg437.png" alt="img" style="zoom:50%;" />
- SafePoint
  - 以“是否具有让程序进入长时间运行的特征”作为标准，如方法跳转、异常跳转
  - GC 过程中用户线程的中断方式是主动式中断，具体行为是用户线程不断轮训收集器的中断标志，如果为真则就近的安全点中断自身
  - 对于已挂起的用户线程，其在挂起之前，先将自身标记为“已进入安全域”（Safe Region），解决用户线程在 Sleep、Blocked 时无法响应 JVM 中断请求而导致 JVM 等待的问题
- TLAB 的全称是 Thread Local Allocation Buffer，默认给每个线程开辟一个 buffer 区域（Eden 区），加速对象分配。

### 类加载

- 类的生命周期

  <img src="0JavaSummary.assets/image-20210712083625754.png" alt="image-20210712083625754" style="zoom: 33%;" />

- 类的加载顺序

  - 静态变量 > 静态初始化块 > 成员变量 > 初始化块 > 构造器

- 双亲委派模型：

  - 委托给父类加载器加载，防止内存中同一个对象出现多次。
  - <img src="0JavaSummary.assets/image-20210712084136084.png" alt="image-20210712084136084" style="zoom: 50%;" />
  - BootStrap ClassLoader，加载/JAVA_HOME/lib下的类库，或-Xbootclasspath指定的路径且能被JVM识别的类库
  - Extension ClassLoader，加载/JAVA_HOME/lib/ext下的类库，或java.ext.dirs系统变量指定路径下的类库
  - Application ClassLoader，加载用户类路径上（classpath）的指定类库
  - 类加载器的作用：判断是类是否相等

### Card Table

- 解决问题：分代收集中跨代引用，引入卡表后只需扫描对应的 Dirty Card
- CMS  Remember Set ，实现就是 Card Table
- ![img](0JavaSummary.assets/120188.png)

- Card Page
  - 将堆空间划分为2次幂大小的卡页，512字节
  - 卡表：标记每个 Card Page 的状态
- Write Barrier
  - 对象引用发生写操作，写屏障将标记 Card Page 标记位 dirty。（CMS和G1）

- Remember Set概念
  - Card Table 是 remember set 的一种具体实现（字长精度、对象精度、卡精度）

### 三色标记法

- 解决问题：标记清除算法，存在长时间暂停。（实时性要求高的系统不能用）

- 异步执行，以极少的中断时间代价来进行整个 GC。

- 三色标记

  - 黑色：根对象，或者该对象与它的子对象都被扫描
  - 灰色：对象本身被扫描，但还没扫描完该对象中的子对象
  - 白色：未被扫描对象，扫描完成所有对象之后，最终为白色的为不可达对象，即垃圾对象

- 标记过程

  * 根对象为黑，其子对象为灰
  * 遍历灰对象，将白的变灰，灰的变黑
  * 遍历完，只剩下黑和白

- 存在的问题

  - 标记过程不是原子性的，因此对象引用关系变化会导致多标漏标问题（细节TODO）

    ![image-20210712092834113](0JavaSummary.assets/image-20210712092834113.png)

  - 漏标问题：当A被置为黑色（完成标记），B被置为灰色等待扫描子对象时，子对象C的引用关系由B->C变成A->C，就会导致C漏标

* 解决：
  * Incremental update（CMS）
    - 简要：只要在 write barrier 里发现要有一个白对象的引用被赋值到一个黑对象的字段里，那就把这个白对象变成灰色的（例如说标记并压到 marking stack 上，或者是记录在类似 mod-union table 里）
    - post-write barrier
  * SATB（G1）
    - 简要原理：把 marking 开始时的逻辑快照里所有的活对象都看作活的，具体做法是在 write barrier 里把所有旧的引用所指向的对象都变成非白的（已经黑灰就不用管，还是白的就变成灰的）
    - 解决的问题：CMS 重新标记阶段暂停时间过长的风险
    - 通过 TAMS 指针识别并发 GC 过程中新分配的对象，新分配的都认为的活的对象（隐式标记）
    - <img src="0JavaSummary.assets/120150.png" alt="img" style="zoom:50%;" />
      - 第 n 轮并发标记开始，Region 当前的 top 指针赋值给 next TAMS，在并发标记标记期间，新对象都在[next TAMS, top]之间分配，SATB 确保这部分的对象都会被标记，默认都是存活的
      - 当并发标记结束时，next TAMS 所在的地址赋值给 previous TAMS，SATB 给 [bottom, previous TAMS] 之间的对象创建一个快照 Bitmap，垃圾对象通过快照被识别出来
  * pre-write barrier
    - 拦截对象引用修改写入操作，通 过G1SATBCardTableModRefBS::enqueue(oop pre_val) 把原引用保存到 satb mark queue 中，最终这部分会被合并的 Snapshot 中

### 收集器原理机制

#### Concurrent Mark Sweep

- 概念、数据结构

  - DirtyCard
  - Mod-Union Table（具体实现是Bitmap，标识新生代晋升、直接在老年代分配、老年代引用关系变更的对象）
  - RememberSet，记录老年代哪个Card中的对象引用了新生代（O -> Y），解决新生代标记的问题
  - PromotionFailed、Concurrent Mode Failure

- 回收步骤

  ![image-20210712093543873](0JavaSummary.assets/image-20210712093543873.png)

  - CMS-initial-mark, STW，初始标记 GC-Roots 可达对象：遍历新生代对象，标记可达的老年代对象；默认单线程，可通过配置调整为多线程（-XX:+CMSParallelInitialMarkEnabled）

  - CMS-concurrent-mark，遍历 initial-mark 标记的对象，递归标记这些对象可达的对象，针对老年代引用关系变更记录 Dirty Card，新生代对象晋升记录 Mod-Union Card（若某个CardTable中的Card中记录为1，YoungGC 时扫描该 Card 发现没有持有新生代的引用，那么该 Card 清除，并将 Mod-Union Card 中对应元素置为 1）

    - CMS-concurrent-clean，optional，默认开启；处理因为上阶段过程中，引用关系改变，未标记的对象变成存活的，会扫描Dirty的Card。如下图的的3和6，在上阶段是未标记对象，即不可达对象。

    <img src="0JavaSummary.assets/image-20210712104338118.png" alt="image-20210712104338118" style="zoom: 50%;" />

    - CMS-concurrent-abortable-preclean， Optional ，承担下一个阶段Final Remark阶段足够多的工作，期待能够发送一次YoungGC。若 Eden 区 CMSScheduleRemarkEdenSizeThreshold=2M，则略过此步骤；否则循环执行 concurrent-mark ，直到 1）达到设置的循环次数（默认0），2）达到执行时间限制（默认5s），Eden 区内存使用率达到阈值 CMSScheduleRemarkEdenPenetration（默认50%）；可通过 CMSScavengeBeforeRemark 配置每次 abortable-preclean 都触发一次 Young GC。

  - CMS-final-remark，STW，**标记整个老年代的所有的存活对象**。具体：遍历新生代对象重新标记，根据老年代GC Roots重新标记，遍历老年代 Dirty Card 重新标记（大部分 Dirty Card 已经在 clean 阶段处理过）。（耗时长，则提前触发一次YoungGC）

  - CMS-concurrent-sweep，回收不可达对象，三种情况下会触发压缩：

    - UseCMSCompactAtFullCollection (默认true)和CMSFullGCsBeforeCompaction(默认0)时每次GC都进行压缩（其实是整理）
    - 执行了System.gc()
    - 新生代分配担保失败

  - CMS-concurrent-reset，重置CMS内部的数据结构，进入下一个CMS生命周期

- 存在的问题

  - 长时间运行内存碎片化
  - final remark存在风险，停顿时间可能过长
  - 大内存性能差，GC时间不可控

#### G1

标记-整理，局部（两个 Region 之间）“复制”，无内存空间碎片。

- 几个重要的数据结构：
  - Region、CSet（CollectionSet）多个 Region 构成回收集
  - G1 Remembered Set：记录本Region中所有对象引用的对象所在的区域（我指向谁，谁指向我），防止全堆扫描

![image-20210712133452256](0JavaSummary.assets/image-20210712133452256.png)

* 初始标记（Initial Marking)：STW，一条初始标记线程对所有与 GC Roots 直接关联的对象进行标记。触发一次Mintor GC。
* 并发标记(Concurrent Marking)：使用**一条**标记线程与用户线程并发执行。速度很慢。此外，当 对象 图 扫描 完成 以后， 还要 重新 处理 SATB 记 录下 的 在 并发 时有 引用 变动 的 对象。
* 最终标记（Final Marking）：STW。再标记阶段是用来收集 并发标记阶段 产生新的垃圾(并发阶段和应用程序一同运行)；G1中采用了比CMS更快的初始快照算法:snapshot-at-the-beginning (SATB)。
* 筛选回收（Live Data Counting and Evacuation）：STW，统计Region数据，回收价值和成本排序，根据期望停顿时间来回收。回收单元是Collection Set，复制到空Region，多线程。



- 对象划分的规则
  - 小于一半Region, Eden
  - 大于一半, Humongous
  - 超过一个Region, Humongous连续Region

**另外一种源码阐述**

- 每次回收都只回收 CSet（Collection Set）中的 Region ，YoungGC 即 CSet 中只包含 young region 、 MixedGC 则是 Cset 中包含 young region 和 old region
- 混合 GC 的触发条件：触发阈值 -XX:InitiatingHeapOccupancyPercent(默认45%)
- 混合 GC 的步骤
  - Initial Mark，STW，借助一次 YoungGC 完成，标记可能持有老年代对象引用的Survivor Region，同时初始化TAMS指针用于记录新分配对象
  - Root Region Scanning，并发执行，扫描 Survivor 区根引用，必须在下一次 YoungGC 到来之前完成
  - Concurrent Marking，并发执行，寻找整个堆空间中存活的对象，可被一次YoungGC中断
  - Remark，STW，完成最终的整堆存活对象标记，使用 SATB 算法
  - Cleanup，统计存活对象和完全空闲的 Region（STW)，擦除 RSet 内容（STW），重置空的 Region 并将其返还给 free list（并发）
  - Copying，STW，拷贝存活对象到新的未使用的 Region 空间，可以是 YoungGC，也可以是 MixedGC 时发生

#### SATB

混合GC使用的标记算法：Snapshot at the Begining

- ConcurrentG1RefineThread，只专注扫描日志缓冲区记录的卡片来维护更新RSet

- pre-write barrier，post-write barrier

- logging write barrier

  - SATBMarkQueue，SATBMarkQueueSet（pre-write barrier）
  - DirtyCardQueue，DirtyCardQueueSet（post-write barrier，解决的是RSet指向关系变更的问题）
  - 为减少 write barrier 对 mutator 的性能影响，G1 收集器将部分 barrier 的逻辑记录到队列（SATBMarkQueue、DirtyCardQueue）中，再由其他线程消费队列批量处理

- 简要原理：把 marking 开始时的逻辑快照里所有的活对象都看作活的，具体做法是在 write barrier 里把所有旧的引用所指向的对象都变成非白的（已经黑灰就不用管，还是白的就变成灰的）

- 解决的问题：CMS 重新标记阶段暂停时间过长的风险

- 通过 TAMS 指针识别并发 GC 过程中新分配的对象，新分配的都认为的活的对象（隐式标记）

- TAMS（top at mark start），previous TAMS，next TAMS

  ![img](0JavaSummary.assets/120135.png)

  - 第 A 步：初始标记阶段，需要 STW，将扫描 Region 的 Top 值赋值给nextTAMS
  - 第 A ~ B 步之间：会发生并发标记阶段
  - 第 B 步：重新标记阶段，此时并发标记阶段生成的新对象都会被分配在 [nextTAMS, Top] 之间，这些对象会被定义为“隐式对象”，同时 _next_mark_bitmap 也开始存储nextTAMS标记的对象的地址
  - 第 C 步：清除阶段，_next_mark_bitmap 和 _prev_mark_bitmap 会进行交换，同时清理 [Bottom, previousTAMS] 之间被标记的所有对象，对于“隐式对象”会在下次垃圾收集过程进行回收（如第 F 步），这也是 SATB 存在弊端，会一定程度产生未能在本次标记中识别的浮动垃圾

- 存在的问题
  - 要维护数量众多的跨 Region 引用，需要复杂的卡表，消耗大量内存；CMS 只需一张老年代到新生代的卡表
  - 需要写前屏障来维护大量卡表，还需要写后屏障来维护原始快照

### 调优

- 如何判断一个程序是否正常？如何评价垃圾收集器的性能好坏呢？
  - 吞吐量：应用程序耗时，GC耗时，不低于95%
  - 停顿时间
  - 垃圾回收频率
-  降低 Minor GC 频率
  - 短对象多，扩容 Eden
  - 长对象多，谨慎扩容

- 单次停顿过长
  - Xmx、Xms
  - AlwayPretouch、Swap、Cpu load
  - Concurrent GC Thread
- Full GC频率较高，整体吞吐率低
  - 减少创建大对象，业务优化
  - 增大堆
- 回收率较低（新生代、老年代）
- 导致 FullGC 的失败
  - CMS Failure：PromotionFailed、ConcurrentModeFailure
  - G1 Failure：EvacuationFailure、Humongous Object Fragmentation
- G1：MixedGC 慢、UPdateRS、ScanRS 慢、Object Copy 慢
- MixedGC 调优：
  - -XX:G1MixedGCCountTarget，一次并发标记之后，最多执行 Mixed GC 的次数。增加次数，降低单次延迟
  - -XX:G1MixedGCLiveThresholdPercent，避免将较满的 Region 加入候选
  - -XX:G1HeapWastePercent，增加堆的冗余度，老年代的垃圾占比5%，超过阈值，mixed GC
- 更造触发 GC 避免单次GC停顿过长
  - -XX:-G1UseAdaptiveIHOP and -XX:InitiatingHeapOccupancyPercent
- sys、user、real



![img](0JavaSummary.assets/119900.png)

**详细版**

- -XX:+AlwaysPreTouch，启动的时候真实的分配物理内存给JVM
  - 新生代对象晋升，要为老年代先分配物理内存，影响了新生代GC的效率。
  - 优点：加快代码运行效率，缺点：启动时间变慢。



### ZGC

- 目标
  - STW不超 10ms
  - 不管多大的堆都能保持在 10ms 以下
  - 最大支持 4T堆

- 概述
  - 内存分成一个个 page，清理压缩
  - 直接利用对象的引用指针，用来标识对象的状态

## 4 分布式协议

- CAP

  - 一致性（各节点间的数据一致）、可用性（**有限时间**内**返回结果**）、分区容错性（部分子网络故障不会导致整个系统不可用）
    - 一致性分为
      - 强一致性
      - 单调一致性 
      - 会话一致性
      - **最终一致性**：用户只能读到某次更新后的值，但系统保证数据将最终达到完全一致的状态，只是所需时间不能保障。
      - 弱一致性
      - 顺序一致性：ZAB
  - Zookeeper，CP 。分析A:极端情况下，不能保证每次服务请求的可用性；leader选举时集群都是不可用。ZK主要是存储Kafka元数据信息，强一致性就很重要。
  - Kafka，CA。
    - 分区读写由leader负责，满足Consistency原则。
    - 分区副本机制，保证可用性A。副本分区数据与leader存在差别怎么办，怎么解决P？
    - 尽量保持分区容错性：ISR的同步策略
  - Eureka：AP
  - Redis：AP，缓存，如果容忍读到过期的数据，那么C就不能满足。
  - Mysql：CA，单机
  - 

- BASE

  - 核心：基本可用（Basically Available）和最终一致性（Eventually consistent）
  - **基本可用的方法：流量削峰、延迟响应、体验降级、过载保护**
  - Basically Available（响应时间增加、功能部分损失如降级）、Soft state（允许不同节点数据副本之间进行数据带来的延迟）、Eventually consistent（数据经过一段时间同步后最终能到达一个一致的状态）

  实现最终一致性的具体方式？

  - 读时修复：在读取数据时，检测数据的不一致，进行修复。

  - 写时修复：在写入数据，检测数据的不一致时，进行修复。

  - **异步修复：这个是最常用的方式，通过定时对账检测副本数据的一致性，并修复。**

### 2PC

- 阶段1 提交事务请求（投票阶段）：
  - 事务询问（proposer）
  - 执行事务，记录undo、redo（acceptor）
  - 各参与者向协调者反馈事务询问响应（acceptor）
- 阶段2 执行事务提交
  - 假如所有参与者反馈都是yes，则执行事务提交
    1. 发送commit请求（proposer）
    2. 事务commit（acceptor）
    3. 反馈事务commit结果（acceptor）
    4. 完成事务（proposer）
  - 假如任何一个参与者反馈no，则执行中断事务
    1. 发送rollback请求（proposer）
    2. 事务rollback（acceptor）
    3. 反馈事务rollback结果（acceptor）
    4. 中断事务（proposer）
- 存在的问题：同步阻塞、单点问题、脑裂（致使数据不一致）、过于保守（少量节点失败导致整体提交失败）

### 3PC

- 改进2PC的提交事务请求阶段，变成三个阶段：

  - CanCommit
    1. 事务问询（proposer）
    2. 各个参与者向协调者反馈事务询问的响应（acceptor）
  - PreCommit
    1. 发送预提交请求（proposer）
    2. 事务预提交（acceptor）
    3. 各参与者向协调者反馈事务预执行的相应ack/abort（acceptor）
    4. 若任何一个参与者反馈no，则中断事务（proposer）
       - 发送中断请求abort（proposer）
       - 中断事务（acceptor）
  - do Commit，一旦进入此阶段，即使协调者出现问题或网络故障，参与者都会在等待超时之后继续提交事务
    1. 发送提交请求（proposer）
    2. 事务提交（acceptor）
    3. 反馈事务提交结果（acceptor）
    4. 完成事务（proposer）
    5. 若协调者正常工作且任意一个参与者反馈no，则中断事务（proposer）
       - 发送中断请求（proposer）
       - 事务回滚（acceptor）
       - 反馈事务回滚结果（acceptor）
       - 中断事务（proposer）

- 优缺点

  1. 将commit确认和实际commit拆分，降低了参与者阻塞范围，单点故障后，仍然可继续达成一致
  2. 但是preCommit阶段网络故障依然会导致不一致问题（TODO)

  

### Paxos

-  Basic Paxos 算法，描述的是多节点之间如何就某个值（提案 Value）达成共识；
-  Multi-Paxos 思想，描述的是执行多个 Basic Paxos 实例，就一系列值达成共识。



- 三个角色：Proposer、Acceptor、Learner
- 通过不断加强这个约束：“在一次Paxos算法执行实例中，只批准一个value”，获得了 Paxos 算法
- 论文里，Acceptor保证**三个重要的承诺**：
  - 如果准备请求的提案编号，**小于等于**接受者已经响应的准备请求的提案编号，那么接受者将承诺不响应这个准备请求；
  - P1a: 如果接受请求中的提案的提案编号，**小于**接受者已经响应的准备请求的提案编号，那么接受者将承诺不通过这个提案；
  - P2c: 如果接受者之前有通过提案，那么接受者将承诺，会在准备请求的响应中，包含**已经通过的最大编号的提案信息**。

**Basic Paxos算法流程**

<img src="0JavaSummary.assets/image-20210712140352312.png" alt="image-20210712140352312" style="zoom:67%;" />

- 算法伪代码

<img src="0JavaSummary.assets/120555.jpeg" alt="img" style="zoom:67%;" />

- proposer 生成提案

  - prepare 阶段

    - proposer 选择一个新的提案编号 n ，然后发送请求给 acceptor 集合，并要求其作如下承诺：

      - acceptor 收到 prepare 请求后，如果提案的编号大于它已经回复过的所有 prepare 消息(**回复消息表示接受  accept**)，则 acceptor 将自己上次接受的提案回复给 proposer，并承诺不再回复小于 n 的提案

      - 举例说明：

        假设一个 acceptor 已经响应过（accept）所有的 prepare 请求，对应提案编号为 1、2、3、...、7，那么 acceptor 接收到编号为 8 的 prepare 请求会，就会将编号为 7 的提案（7，value=x）作为响应反馈给 proposer 

  - accept 阶段

    - 当一个 proposer 收到了多数 acceptors 对 prepare 的回复后，就进入批准（accept）阶段 
    - proposer 要向回复 prepare 请求的 acceptors 发送 accept 请求，包括编号 n 和根据 P2c 决定的 value（如果根据P2c没有已经接受的value，那么它可以自由决定value） 
    - 只要 acceptor 尚未对编号大于 n 的 prepare 请求响应，就可以通过这个提案 

  - learner 提案获取（解决单个proposer向大量节点同步数据引起的性能问题）

    - 选取一批 learner 集合作为主 learner 集，acceptor 将批准的提案发送给这个集合，这个集合的每个 learner 可以在一个提案被选定后通知所有其他的 learner 

#### 优缺点

- 优点，具有容错能力：当少于一半的节点出现故障的时候，共识协商仍然在正常工作。
- 局限性：只能就单个值（Value）达成共识。
- 活锁问题：多个Proposer的问题
  - proposer-1 提出 n1，完成了阶段1(准备阶段) 
  - 此时 proposer-2 提出 n2，也完成了阶段1
  - 由于提案号n2>n1,于是 acceptor 忽略 proposer-1 发送的 accept 请求，这导致 proposer-1 再次进入阶段1（伪代码第6步）并提出 n3(n3>n2)，而如果它也完成了阶段1，就会导致  proposer-2 在阶段2的 Accept 请求被忽略 。
  - 以此类推，提案选定过程将陷入活锁
- 2轮RPC问题

### Multi-Paxos

直接通过多次执行 Basic Paxos 实例，来实现一系列值的共识。

- Basic Paxos只能对一个值形成决议，决议的形成至少需要2 轮 RPC 通讯，在高并发情况下需要更多的网络来回可能形成活锁 
- Basic Paxos 无法支持连续确定多个值，因此 Basic Paxos 不适合应用在实际工程中 

- Multi-Paxos 正是为解决此问题而提出，Multi-Paxos 基于 Basic Paxos 做了两点改进: 
  1. 针对每一个要确定的值，运行一次 Paxos 算法实例（Instance），形成决议，每一个 Paxos 实例使用唯一的 Instance ID 标识 
  2. 在所有 Proposers 中选举一个 Leader，由 Leader 唯一地提交 Proposal 给Acceptors 进行表决。
     - Multi-Paxos 首先需要选举 Leader，可执行一次 Basic Paxos 实例来选举出一个  Leader 
     - 选出 Leader 之后只能由 Leader 提交 Proposal，没有  Proposer 竞争，解决了活锁问题
     - 在系统中仅有一个 Leader 进行 Proposal 提交的情况下，Prepare 阶段可以跳过 ，两阶段变为一阶段，提高效率 

**改进点：**

- 领导者节点作为唯一提议者。

- 优化 Basic Paxos 执行
  - “当领导者处于稳定状态时，省掉准备阶段，直接进入接受阶段”

整个流程：

![image-20210706145658776](0JavaSummary.assets/image-20210706145658776-1625554619751-6070133.png)

## Raft

**Raft 算法是通过一切以领导者为准的方式，实现一系列值的共识和各节点日志的一致。**

> Raft 算法属于 Multi-Paxos 算法，做了一些简化和限制，比如增加了日志必须是连续的，只支持领导者、跟随者和候选人三种状态。**现在分布式系统开发首选的共识算法**，比如 Etcd、Consul

如何保证在同一个时间，集群中只有一个领导者呢？

服务器节点状态有3种：

- 领导者（Leader）
- 跟随者（Follower）: 接收和处理来自领导者的消息，当等待领导者心跳信息超时的时候，就主动站出来，推荐自己当候选人。
- 候选人（Candidate）: 向其他节点发送请求投票（RequestVote）RPC 消息，通知其他节点来投票，如果赢得了大多数选票，就晋升当领导者。
- ![img](0JavaSummary.assets/1089769-20181216202049306-1194425087-5929203-6070133.png)

### 0 请求完整流程

  当系统（leader）收到一个来自客户端的写请求，到返回给客户端，整个过程从leader的视角来看会经历以下步骤：

- leader **append log entry**
- leader issue AppendEntries RPC in parallel (日志复制)
- leader wait for majority response, **committed**
- leader **apply entry to state machine**
- leader reply to client
- leader notify follower apply log

> Raft中，副本数据是以日志的形式存在的，领导者接收到来自客户端写请求后，处理写请求的过程就是一个复制和提交日志项的过程。

### 1 选举领导者

是属于Basic Paxos。

- election timeout（wait until become candidate）
  - 初始化在 150-300ms 之间，follower 等待超时后会成为 candidate 状态
  - follower 成为 candidate 后就会发起一次新的选举任期，它会给自己投票，并向其他 node 发送 “Request Vote” 消息
  - 接收到请求的node如果尚未投票且在此轮选举中，则直接投票给 candidate，并重置 election timeout
- heartbeat timeout（heatbeat send interval）
  - Leader定期向follower发送“Append Entries”消息，发送间隔为heartbeat timeout
  - Follower响应Append Entries消息，并重置election timeout
  - 整个状态不断维持，直到一个follower不再收到heartbeat

- （全部是跟随者）初始状态下，集群中所有的节点都是跟随者的状态。

<img src="0分布式协议.assets/image-20210706151804779.png" alt="image-20210706151804779" style="zoom: 50%;" />

> Raft 算法实现了随机超时时间的特性。

#### 细节

**节点间是如何通讯的呢？**RPC，2类RPC

- （候选人）请求投票（RequestVote）RPC，是在选举期间发起，通知各节点进行投票；

- （领导者）日志复制（AppendEntries）RPC，是由领导者发起，用来复制日志和提供心跳消息。

**选举有哪些规则？**

- **领导者发送心跳消息**（即不包含日志项的日志复制 RPC 消息），阻止跟随者发起新的选举。
- 如果心跳超时，跟随者推举自己为候选人，发起领导者选举。
- 每一个服务器节点最多会对一个任期编号投出一张选票，“先来先服务”。
- 在选举中，赢得大多数选票的候选人，将晋升为领导者。

（如何保证任何时候只有1个领导者，如何减少选举失败?）

- 任期
- 领导者心跳消息
- 随机选举超时
- 先来先服务
- 大多数选票原则

对先来先服务的补充：

- 当任期编号相同时，日志完整性高的跟随者（也就是最后一条日志项对应的任期编号值更大，索引号更大），拒绝投票给日志完整性低的候选人。

![image-20210706153754030](0分布式协议.assets/image-20210706153754030-1625557075118.png)



**随机超时时间又是什么？**

解决选举过程中，选票被平均瓜分，导致选举无效的情况发生。（随机超时，可以让大多数情况下，只有1个节点先发起选举)

- 心跳信息超时的时间间隔，是随机的；

- 当没有候选人赢得过半票数，选举无效了，这时需要等待一个随机时间间隔，也就是说，等待选举超时的时间间隔，是随机的。

#### 与Multi-Paxos不同

- Raft不是所有的节点都可以是Leader，只有日志最完整的节点才可以。
- 日志必须是连续的。Multi-Paxos 不要求日志是连续的



### 2 复制日志

共识算法的实现一般是基于复制状态机（Replicated state machines）(**相同的初识状态 + 相同的输入 = 相同的结束状态**。)

- 在raft中，leader将客户端请求（command）封装到一个个log entry，将这些log entries复制（replicate）到所有follower节点，然后大家按相同顺序应用（apply）log entry中的command，则状态肯定是一致的。

  <img src="0JavaSummary.assets/1089769-20181216202234422-28123572-6070133.png" alt="img" style="zoom:67%;" />



<img src="0分布式协议.assets/image-20210706165109245-1625561470718.png" alt="image-20210706165109245" style="zoom:50%;" />

**如何复制日志？优化 Basic Paxos 执行**

- 领导者进入第一阶段，通过日志复制（AppendEntries）RPC 消息，将日志项复制到集群其他节点上。
- 如果领导者接收到大多数的“复制成功”响应后，它将日志项提交到它的状态机，并返回成功给客户端。如果领导者没有接收到大多数的“复制成功”响应，那么就返回错误给客户端。

> 领导者提交了自己的日志项，为什么没有通知跟随者提交日志项呢?
>
> 这是Raft的一个优化，额外通过其他时候的日志复制 RPC 消息或心跳消息告知。因为这些消息里面包含了当前最大的，将会被提交的日志项索引值。
>
> - 好处：降低了一半的消息延迟



#### **如何复制日志的具体流程？**

![image-20210706170810690](0JavaSummary.assets/image-20210706170810690-1625562491796-6070133.png)

- Leader接收到客户端请求后，创建一个新日志项（包含指令），写到本地日志。

- Leader日志复制 RPC，通知其他Follower。

- Leader收到大多数复制成功，日志项提交到状态机中。

- Leader返回客户端。

- Follower接收到心跳信息，或者日志复制 RPC 后，如果Follower发现Leader提交了某条日志项，Follower提交自己的状态机。

#### 如何实现日志的一致？

源于：进程崩溃、服务器宕机等问题

思路：Leader通过强制Follower直接复制日志项，处理不一致日志。具体有 2 个步骤。

- Leader通过日志复制 RPC 的一致性检查，找到跟Follower，与自己相同日志项的最大索引值。
- 强制Follower更新覆盖的不一致日志项，实现日志的一致。

![image-20210706171730627](0分布式协议.assets/image-20210706171730627-1625563051916.png)



- 领导者发送当前最新日志项到跟随者（7,4)

- 跟随者在它的日志中，找不到(7,4) 的日志项，跟随者返回失败信息。

- 领导者会递减日志项，并发送新的日志项（6,3）

- 跟随者在它的日志中，找到了 （6,3），返回成功。

- 领导者通过日志复制 RPC，复制并更新覆盖跟随者。（实现了日志一致性）

> 如何解决leader频繁切换导致的日志可能被回滚的问题？
>
> 某个leader选举成功之后，不会直接提交前任leader时期的日志，而是通过提交当前任期的日志的时候“顺手”把之前的日志也提交了，具体怎么实现了，在log matching部分有详细介绍。那么问题来了，如果leader被选举后没有收到客户端的请求呢，论文中有提到，在任期开始的时候发立即尝试复制、提交一条空的log

#### 小结

- 在 Raft 中，副本数据是以日志的形式存在的，指令表示用户指定的数据。
- Multi-Paxos 不要求日志是连续的，但在 Raft 中日志必须是连续的。日志完整性最高的节点才能当选领导者。
- Raft 是通过以领导者的日志为准，来实现日志的一致的。

### 3 成员变更

背景：节点上下线，原有3台节点，现在新入2台，如何保证集群不会分裂，出现2个领导者。

**最常用的方法：单节点变更**。

- 有什么办法能突破 Raft 集群的写性能瓶颈呢？参考Kafka的分区和ES的主分片副本分片这种机制



#### **网络分区**

怎么保证读写数据的正确性？

<img src="0JavaSummary.assets/1089769-20181216202652306-2050900084-6070133.png" alt="img" style="zoom:50%;" />

在系统中貌似出现了两个leader：term 1的Node B， term 2的Node E, Node B的term更旧，但由于无法与Majority节点通信，NodeB仍然会认为自己是leader。

在这样的情况下，我们来考虑读写。

- 写请求发送到了NodeB，NodeB无法将log entry 复制到majority节点，因此不会告诉客户端写入成功，这就不会有问题。
- 读请求，stale leader可能返回stale data，比如在read-after-write的一致性要求下，客户端写入到了term2任期的leader Node E，但读请求发送到了Node B。如果要保证不返回stale data，leader需要check自己是否过时了，办法就是与大多数节点通信一次，这个可能会出现效率问题。另一种方式是使用lease，但这就会依赖物理时钟。（TODO）
- 从raft的论文中可以看到，leader转换成follower的条件是收到来自更高term的消息，如果网络分割一直持续，那么stale leader就会一直存在。而在raft的一些实现或者raft-like协议中，leader如果收不到majority节点的消息，那么可以自己step down，自行转换到follower状态。

#### leader crash

leader在请求过程中，任一时候crash，raft是如何容错的，保障数据一致性的？

当leader crash的时候，事情就会变得复杂。在[这篇文章](http://www.cnblogs.com/mindwind/p/5231986.html)中，作者就给出了一个更新请求的流程图。

<img src="0分布式协议.assets/image-20210710233035986.png" alt="image-20210710233035986" style="zoom:50%;" />

记录一下重要的：

- 3.1阶段：数据到达 Leader 节点，成功复制到 Follower 所有节点，但还未向 Leader 响应接收
  - 这个阶段 Leader 挂掉，虽然数据在 Follower 节点处于未提交状态（Uncommitted）但保持一致，重新选出 Leader 后可完成数据提交，此时 Client 由于不知到底提交成功没有，可重试提交。针对这种情况 Raft 要求 RPC 请求实现幂等性，也就是要实现内部去重机制。
- 3.1 阶段：数据到达 Leader 节点，成功复制到 Follower 部分节点，但还未向 Leader 响应接收
  - 这个阶段 Leader 挂掉，数据在 Follower 节点处于未提交状态（Uncommitted）且不一致，Raft 协议要求投票只能投给拥有最新数据的节点。所以拥有最新数据的节点会被选为 Leader 再强制同步数据到 Follower，数据不会丢失并最终一致。
- 网络分区导致的脑裂情况，出现双 Leader
  网络分区将原先的 Leader 节点和 Follower 节点分隔开，Follower 收不到 Leader 的心跳将发起选举产生新的 Leader。这时就产生了双 Leader，原先的 Leader 独自在一个区，向它提交数据不可能复制到多数节点所以永远提交不成功。向新的 Leader 提交数据可以提交成功，网络恢复后旧的 Leader 发现集群中有更新任期（Term）的新 Leader 则自动降级为 Follower 并从新 Leader 处同步数据达成集群数据一致。

- 领导者节点作为唯一提议者。

### 4种协议对比

<img src="0JavaSummary.assets/121186.png" alt="img" style="zoom:120%;" />

## ZAB

### 0定义

- 原子一致性协议，属于顺序一致性

- 2种角色，3种状态
  - leader：接受所有客户端请求
  - follower：处理客户端非事务请求，参与Proposal的投票和leader选举。
  - Leading：主服务节点状态
  - Following：从服务节点状态
  - Looking：选举状态
- ZXID:Epoch e和序列号c
  - ZXID 低 32 位为自增计数，高 32 位代表了 Leader 周期 epoch 编号
  - 新的 Leader ，用最大事务提议的 ZXID，解析epoch，然后+1，作为新的ZXID

- transactions：leader 将客户端变更传播给 follower（TODO）
- 'e'：Leader 的 epoch（任期、term等类似概念）
- 'c'： Leader 生成的序列号，单调递增. 这个值和 epoch （二者通过位运算共同构成 ZXID ）
- 'F.history'：Follower 的历史队列，提交按顺序接收到的事务
- 未提交的事务：F.history 中的事务，序列号小于当前提交的序列号

- ZAB 所需前提
  1. 复制担保
     - 可靠投递：如果事务 M 被一台服务器提交（commit），其终将被所有服务器提交
     - 全局有序：如果事务 A 于事务 B 之前被一台服务器提交，在所有其他服务器上 A 也将先于B被提交. 如果 A 和 B 都被提交，要么 A 先于 B 被提交，要么 B 先于 A 被提交（不存在同时提交）
     - 因果有序：如果事务 B 在事务 A 提交之后被 sender B 发送，那么 A 的顺序必须在 B 之前. 如果一个sender先发送 B 后发送 C，那么 C 的顺序必须排在 B 之后.（逻辑顺序不受物理条件影响，比如发送端的顺序和接收端最终提交的顺序必须一致）
  2. 只要大多数（quorum）nodes 已启动，事务就会被复制
  3. 若 node 故障但随后又重启，它应能追上故障期间已经复制完成的事务

### 1消息广播复制

- Client可读任意节点
- Client写请求会被转发给Leader
- **2PC变种来实现日志复制**
  1. leader 接收写请求，生成序列号 c 和 leader epoch的事务，发送所有Follower
  2. Follower 将事务添加 history queue ,回复ACK
  3. leader 接收到大多数ACK ，发送 commit 请求
  4. Follower 提交事务
     - 保证顺序性的前提：Follower等待序列化号小的事物处理完成后，提交当前的事务 

<img src="0JavaSummary.assets/120353.png" alt="img" style="zoom:67%;" />

### 2集群崩溃恢复

 leader 崩溃，发送leader选举

- **选举过程**

  - **Phase 0：选举（election）**
    - 节点Looking，发起投票，服务器ID和ZXID
    - 其它节点收到请求，对比ZXID，发起请求，投票给大的ZXID
    - 节点统计投票，大多数同意，变成准Leading，其它节点变成Following
  - **Phase 1：发现（discovery）**
    - **准leader**收集节点的epoch值，发送epoch+1
    - follower回复ACK，带上ZXID和历史事务日志（F.History）
    - **准leader** 更新自身的ZXID和事务日志
  - **Phase 2：同步（synchronization）**
    - 准leader发送同步信息
    - 半数Follower同步成功，准Leader成为Leader。
  - **Phase 3：广播（broadcast）**
    - leader接受client写请求
    - 2PC提交：
      - Leader保留提交日志，发送Propose广播给Follower
      - Follower确认，回ACK
      - Leader发送Commit消息，提交事务

  >  Phase 1 和 2 对于集群内的相互一致性很重要，尤其是从故障中恢复时


#### Phase 0 选举

  - 选票数据结构
    - logicClock：每个服务器维护一个自增整数，表示该服务器发起的第几轮投票
    - state：服务器当前状态
    - self_id：服务器的 myid
    - self_zxid：服务器上接收到的事务的最大 zxid
    - vote_id：被推举的服务器 myid
    - vote_zxid：被推举的服务器上保存的事务的最大 zxid
    
  - 投票流程
    - 自增选举轮次
      - 一次有效的投票必须在同一轮次中，开始新一轮投票时，服务器先对自己的logicClock自增
    - 初始化选票
      1. 服务器广播自己的选票前，先将自己的投票箱清空
      2. 投票箱用于记录收到的选票，如：服务器2投票给服务器3，服务器3投票给服务器1，则服务器1的投票箱内将存储(2, 3)，(3, 1)，(1, 1)
      3. 票箱中只记录投票者的最后一票，如投票者A更新自己的选票，其他服务器收到该选票后会在更新票箱中A的选票
    - 发起初始化选票
      - 每个服务器最开始通过广播把票投给自己
    - 接收外部投票
      - 服务器尝试从其他服务器获得投票，计入自己的投票箱内
      - 如果无法获得任何外部选票，则确认自己是否与集群中其他的服务器保持着有效的连接；如果是则再次发送自己的投票；否则马上建立连接
    - 判断选举轮次
      - 收到外部投票后，首先根据投票信息中所包含的 logicClock 来进行不同处理：
        1. 外部投票的 logicClock 大于自身的 logicClock，说明该服务器的选举轮次落后于其他服务器，立即清空自己的投票箱，并把自己的 logicClock 更新为接收到的 logicClock，然后再对比自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去
        2. 外部投票的 logicClock 小于自身的 logicClock，当前服务器直接忽略该选票，继续处理下一个投票
        3. 外部投票的 logickClock 自身的相等，则进行选票 PK
    - 选票PK
      - 选票 PK 基于（self_id, self_zxid）与（vote_id, vote_zxid）的对比
        1. 外部投票的 logicClock 大于自身的，则将自己的 logicClock 及自己的选票的 logicClock 变更为收到的 logicClock
        2. 若 logicClock 一致，则对比二者的 vote_zxid，若外部投票的 vote_zxid 比较大，则将自己的票中的 vote_zxid 与 vote_myid 更新为收到的票中的 vote_zxid 和 vote_myid 并广播出去，另外将收到的票以及自己更新后的票放入自己的票箱. 如果票箱内已存在（self_myid, self_zxid）相同的选票，则直接覆盖
        3. 若二者的 vote_zxid 一致，则比较二者的 vote_myid，若外部的投票的 vote_myid 比较大，则将自己的票种的 vote_myid 更新为收到的票种的 vote_myid 并广播出去，另外将受到的票以及自己更新后的票放入自己的票箱
    - 统计选票
      - 如果已经确定过半服务器认可了自己的投票（可能是更新后的投票），则投票终止；否则继续接受其他服务器的投票
    - 更新服务器状态
      - 投票终止后，服务器开始更新自身状态. 若过半票投给了自己，则将自己的服务器状态更新为LEADING，否则将自己的状态更新为 FOLLOWING

#### Phase 1 发现

（目的：从 quorum 中找到最完备的 F.history）

- **准leader**收集节点的epoch值，发送epoch+1
- follower回复ACK，带上ZXID和历史事务日志（F.History）
- **准leader** 更新自身的ZXID和事务日志
- quorum 做出保证：quorum中至少有一个节点（其 epoch 最大, ZXID最大）的 history queue 是最新的，完整的

<img src="0JavaSummary.assets/120367-1628044937992.png" alt="img" style="zoom:67%;" />

> 注：理论上被选举出来的 prospective leader 应具有最大的 zxid，即接收了最新的事务，为什么还要向 quorum 中的 follower 获取历史事务？



#### Phase 2 同步

（目的：将'发现'步骤中获得的 F.history 作为提案提出）
   - 准leader发送同步信息，将历史事务作为提案
   - 半数Follower同步成功，准Leader成为Leader。
   - follower 自身的事务历史序列落后，认可 leader 
   - 同步完成，恢复（recovery）阶段结束 

<img src="0JavaSummary.assets/120365.png" alt="img" style="zoom:67%;" />

#### Phase 3 广播

- leader接受client写请求
- 2PC提交：
  - Leader保留提交日志，发送Propose广播给Follower
  - Follower确认，回ACK
  - Leader发送Commit消息，提交事务
- 对于 observer，leader 会发送 inform 消息，其中包含提议的内容（follower）

<img src="0JavaSummary.assets/120363.png" alt="img" style="zoom:67%;" />

> 为了检测故障，ZAB 在 leader 和 follower 之间使用定期的心跳消息通信. 如果leader 在一段时间内没有收到 quorum（majority）的心跳，它就放弃自己自己的 leader 身份，将状态切换成选举和 Phase 0. 如果 follower 在一段时间内也没收到 leader 的心跳，就跳转到 Leader Election Phase.

## 4 Zookeeper

* 分布式协调服务框架，主要依靠文件系统和监听通知机制。

- 特性：顺序一致性（主要是写操作的严格顺序性，每个更新请求都会分配一个全局唯一的递增编号）、原子性、单一视图（Single System Image）、可靠性（只要集群中有超过一半的机器能工作整个集群就能对外提供服务）、实时性（ZK将全量数据存储在内存中，因此适合读操作为主的应用场景）

- CP系统，分析A:极端情况下，不能保证每次服务请求的可用性；leader选举时集群都是不可用。

- 三种角色：Leader、Follower、Observer，Leader 提供读写，Follower 和 Observer 只提供读服务，Observer 不参与 Leader 选举过程，也不参与写操作的“过半写成功”策略，因此 Observer 可以不影响写性能的情况下提升集群读性能

  - leader职责是接受所有客户端请求，协调内部各个服务器。 
  - follower职责是处理客户端非事务请求，参与Proposal的投票和leader选举。
  - observer职责是处理客户端非事务请求，不参与投票。（为什么这么设计？）

- session管理(TODO)

  - 三个状态：CONNECTING、CONNECTED、CLOSE
  - SessionID生成

  ```
      long nextSid = 0;
      nextSid = (System.currentTimeMillis() << 24) >> 8;
      nextSid = nextSid | (id << 56);
      return nextSid;
  ```

  - 会话管理采用分同策略，将类似的会话方在同一区块中进行管理，以便ZK对会话进行不同区块的隔离处理以及同一区块的统一处理，分配原则是每个会员的“下次超时时间点”：`ExpirationTime = ((CurrentTime + SessionTimeout) / ExpirationInterval + 1) * ExpirationInterval`

  ![img](0JavaSummary.assets/120061.png)

  - 客户端在会话超时时间过期范围内向服务器发PING保持会话有效性（心跳检测）
    - 服务端则需要接受客户端的PING并按需激活会话（TouchSession），将会话迁移到新的区块内
    - 客户端发现在SessionTimeout / 3的时间内未和服务端通信，则发起一次PING触发服务端激活session
    - 服务端SessionTracker有一个单独的线程专门进行会话超时检查，以ExpirationInterval作为时间点来触发检查，每次检查就检查过期的桶中所有剩下的未被迁移的会话即可
  - 当客户端与服务端网络断开，客户端会自动反复重连直到连上集群中的一台机器，如果在会话超时时间内重新连上，则状态改为 *CONNECTED*（CONNECTION_LOSS），如果超过超时时间才连上，则为 *EXPIRED*（SESSION_EXPIRED）

  

### ZAB Leader选举
见ZAB的集群崩溃恢复

### 脑裂

- 现象：2个leader
  - 假死：由于心跳超时认为Leader死了，但Leader还存活着。
  - 脑裂：假死，发起新的Leader选举。但旧的Leader网络又通了，导致出现了两个Leader 。客户端可以访问到2个leader
- 原因：
  - ZooKeeper集群和ZooKeeper client判断超时并不能做到完全同步
- 影响：
  - 数据不一致
- 常规思路：
  - Quorums（法定人数）方式：半数+1的原则。防止“脑裂”默认采用的方法。
    - 新Leader产生会生成epoch
    - Follower确认了新Leader的存在，拒绝小于epoch的所有请求
    - 旧Leader使用旧的epoch，发出的请求被拒绝
  - Redundant communications（冗余通信）
    - 添加冗余的心跳线，例如双线条线，尽量减少“裂脑”发生机会。
  - Fencing（共享资源）方式
    - 能够获得共享资源的锁的就是Leader，看不到共享资源的，就不在集群中。(分布式锁？)
  - 仲裁机制方式
    - 例如设置参考IP（如网关IP），当心跳线完全断开时，2个节点都各自ping一下 网关IP，不通则表明断点就出在本端，主动放弃竞争，释放共享资源，重启。
    - 能够ping通参考IP可以继续竞争
  - 启动磁盘锁定
    - 正在服务一方锁住共享磁盘，“裂脑”发生时，让对方完全“抢不走”共享磁盘资源。
    - 如果占用共享盘的一方不主动“解锁”，另一方就永远得不到共享磁盘。
    - 假如服务节点突然死机或崩溃，就不可能执行解锁命令。
      - 设计了“智能”锁。即正在服务的一方只在发现心跳线全部断开（察觉不到对端）时才启用磁盘锁。平时就不上锁了。

- 解决方法
  - Follower节点准备切换成Leader时，sleep 超时时间，确保旧Leader完全shutdown。（sleep时，服务不可用，但是数据一致性保证 CP）
  - Quorums（法定人数）方式：半数+1的原则
    - 新Leader产生会生成epoch
    - Follower确认了新Leader的存在，拒绝小于epoch的所有请求
    - 旧Leader使用旧的epoch，发出的请求被拒绝

### 集群故障

- 现象：3.4.9 version,  连接ZK超时，发现Zookeeper Server not running，某个节点重启无法恢复，后面出现整个集群无法启动

```
[myid:5] - WARN  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxn@362] - Exception causing close of session 0x0 due to java.io.IOException: ZooKeeperServer not running
```

- 原因

  - Zookeeper的快照文件snapshot特别大，一个snapshot就6G，并且几分钟就生成一个snapshot。

  - 集群模式下，follower节点需要获取leader节点的snapshot，必须在initLimit时间，否则无法启动Zookeeper

  - ```shell
    tickTime=2000 # 2000ms
    initLimit=10 # The number of ticks that the initial synchronization phase can take
    syncLimit=5 # The number of ticks that can pass between sending a request and getting an ack
    ```

  - syncLimit配置为5，表示sync的timeout有5个tick

  - 比如zk的data数据比较大，在10S内不一定能同步完成，每次zk选举都会同步data,由于syncLimit设置的太短，失败之后再次重新选举，然后再次超时，导致集群不可用

- 解决
  - 调大initLimit
  
  - 调大syncLimit
  
  - ```shell
    autopurge.snapRetainCount=60 
    autopurge.purgeInterval=48
    # 保留48小时内的日志，并且保留60个文件
    ```

​    参考：[Sudden crash of all nodes in the cluster](https://issues.apache.org/jira/browse/ZOOKEEPER-2104?attachmentOrder=desc)

### ZK挂原因

zookeeper集群中leader和follower同步数据的极限值是500M，这500M的数据，加载后，大约占用3G内存

- 数据过大，在每次选举之后，需要从leader同步到follower，2个问题： 
  - 网络传输超时，因为文件过大，传输超过最大超时时间，造成TimeoutException，从而引起重新选举。
  - 如果调大这个超时值，则很可能达到磁盘读写的上限，目前，每像精卫、tbschedule3等，都有大量的zk写入，这些会触发频繁的磁盘写操作，一旦达到io上限值，就会导致超时，进而触发重新选举或直接导致系统崩溃。
  - 导致zookeeper集群在选举和数据同步之间陷入死循环
- 解决思路
  - 加zookeeper server机器提高性能：注意：加zookeeper server机器提高的只是读性能，但机器越多，多写问题就越严重，系统也越容易挂掉。
  - 冗余一个集群，作为当前集群的备份：冗余出来的集群可以比较小，平时并不服务，只有当主集群挂掉时，再自动切换提供服务。这种集群级别的冗余貌似可行，其实也有问题，导致这种方案可行性也不高，关键问题就在于，需要将数据从主集群的leader实时同步到备份集群，这就存在一个IO问题，如果数据量大，异常存在网络超时和IO压力大问题，如果单独提供一个方案实现从主集群leader到备份集群的高速同步，那就可以直接用于解决主集群之间挂掉的问题了，更不需要备份集群了。另外，冗余物理机作为备份，绝大多数情况下，这个物理机都可能是不提供服务的，所以有资源浪费的问题。
  - 细化zk集群：为防止其它业务的影响。如果你的应用强依赖zookeeper，则应该申请机器资源，单独配置zookeeper服务器，防止其他应用的影响。这是目前比较可行的解决方案。
  - 多机房分布：zookeeper存在一个要求，必须有多于n+1台机器存活，否则整个集群挂掉（zookeeper通过这种方式，牺牲了稳定性，但保证了数据一致性）。在目前阿里的双机房策略下，无论两个机房怎么分配，2n+1台机器在两个机房中，都会存在一个机房的机器数大于n+1的问题，如果这个机房挂掉或机房切换，都可能导致整个zk集群挂掉。如果想要保住zk集群稳定性，就必须至少有3个机房，这样一个机房挂掉时，可以保证仍然有n+1台机器存活。这种方案在以后单元化推广开之后还有可能，在目前双机房情况下，无解。
- 建议
  - 能不用zookeeper，就不用zookeeper，如果一定要用，尽量不要强依赖zookeeper；
  - 如果你要用到分布式锁，zookeeper是个不错的选择，如果不需要分布式锁，你应该优先考虑不用zookeeper；
  - 采用监听方式，而不是主动查询方式，相信zookeeper的监听推送吧，只要你实现的代码没问题，它还是很稳定的；
  - 不要对zookeeper频繁写入，它只应该存储控制信息和配置信息，也就是说，它更多应该用来做读操作。
  - 不要把zookeeper作为数据存储器。
  - 不要与那些大应用共用一个zookeeper集群，你可能会被它拖挂的。

- 参考：[Zookeeper一般挂掉的原因](https://www.jianshu.com/p/f30ae8e75d6d)

## 5 Netty

1. Netty 是什么？Netty 的特点是什么？

2. Netty 的优势有哪些？为什么要用  Netty？

3. Netty 的应用场景有哪些？

   

4. BIO、NIO和AIO的区别？NIO的组成？

5. Netty的线程模型？Netty 核心组件有哪些？分别有什么作用？

6. EventloopGroup 了解么?和 EventLoop 啥关系? Bootstrap 和 ServerBootstrap 了解么？



1. NIOEventLoopGroup源码？NioEventLoopGroup 默认的构造函数会起多少线程？
2. Netty 服务端和客户端的启动过程了解么？默认情况  Netty 起多少线程？何时启动？
3. Netty 发送消息有几种方式？



1. Netty 高性能表现在哪些方面？什么是  Netty 的零拷贝？
2. TCP 粘包/拆包的原因及解决方法？
3. 了解哪几种序列化协议？如何选择序列化协议？
4. Netty 支持哪些心跳类型设置？Netty 长连接、心跳机制了解么？
5. Netty 和 Tomcat 的区别？

### IO多路复用

<img src="0JavaSummary.assets/aHR0cHM6Ly91c2VyLWdvbGQtY2RuLnhpdHUuaW8vMjAxOC8xMS8xLzE2NmNjYmJjZmJhNTNjMGE-1623900177294" alt="img" style="zoom:50%;" />

- 多路复用是指单个线程就可以同时处理多个网络连接的IO。
- 原理：select/epoll不断轮询所负责的socket，以注册和监听为基础，当某个socket有数据到达了，就通知用户线程

- poll和epoll的区别，是epoll 只查找注册的感兴趣连接的读写事件，poll每次都查找所有的连接事件
  - 用户线程会阻塞在select方法，java是以epoll的底层的。
- 适用场景：适合连接数多，同时处理多个连接请求

> Java NIO（多路复用IO（IO Multiplexing）：即经典的Reactor设计模式，有时也称为**异步阻塞IO**



- Java NIO概念：Channel、Buffer、Selector。
  - Selector 选择器, 多路复用器（允许单个Selector处理多个 Channel）。作用是：检查多个 Channel（通道）的状态是否处于可读、可写事件。

  - Buffer 缓冲区，一块内存区。（被NIO Buffer包裹起来，对外提供一系列的读写方便开发的接口。）
- Channel 通道，是读写Buffer的入口。Channel 需要向`Selector`注册监听的事件
  - 从通道进行数据读取 ：创建一个缓冲区，然后请求通道读取数据。
  - 从通道进行数据写入 ：创建一个缓冲区，填充数据，并要求通道写入数据。
- **工作原理：Selector负责监听外部事件，Channel把自己注册到Selector，并告诉自己感兴趣的事件。但外部有事件来的时候，就会去轮询Channel，找到合适的Channel来处理。而Buffer是存放数据的地方**

```java

//第一步 创建Selector：通过调用Selector.open()方法创建一个Selector
Selector selector = Selector.open();
//第二步 必须将channel注册到selector上，并且指定感兴趣的事件是 Accept
ssc.register(selector, SelectionKey.OP_ACCEPT);

while (true)
	//第三步 通过Selector选择通道，一旦向Selector注册了多个通道，select()方法返回你所感兴趣的事件（如连接、接受、读或写）已经准备就绪的那些通道。
	int nReady = selector.select(); // 一旦调用了select()方法，并且返回值表明有一个或更多个通道就绪了，然后可以通过调用selector的selectedKeys()方法
	Set<SelectionKey> keys = selector.selectedKeys();

	if (key.isReadable()) 
		// 第四步 SelectionKey.channel()方法返回的通道需要转型成你要处理的类型，如SocketChannel等。
		SocketChannel socketChannel = (SocketChannel) key.channel();
		socketChannel.read(readBuff);
		System.out.println("received : " + new String(readBuff.array()));
```

### 主从Reactor

<img src="0JavaSummary.assets/image-20210726080354467.png" alt="image-20210726080354467" style="zoom: 33%;" />



- mainReactor线程：Acceptor，专门负责建立连接。--bossGroup NioEventLoopGroup
- subReactor 线程池：一个或者多个，专门处理IO请求。--workerGroup NioEventLoopGroup
- worker 线程池：专门处理**非IO请求** -- 具体实现上，和subReactor在同一个线程池中。

### 工作架构

Netty是高性能、**异步事件驱动的NIO**框架，对TCP、UDP和文件传输的支持。

- Selector：**Netty基于Selector对象实现I/O多路复用，通过 Selector, 一个线程可以监听多个连接的Channel事件**
- Channel ，网络I/O操作。**EventLoop** 负责处理注册到其上的**Channel** 处理 I/O 操作。
  - NioEventLoop 中包含了一个 NIO Selector、一个队列、一个线程
  - EventLoopGroup 是线程池实现
- 简要概述Reactor架构
  - BossGroup：处理TCP连接。一个线程作为MainReactor，处理accept事件
    - 接收客户端TCP连接，把事件任务放到TaskQueue
    - NioEventLoop Thread处理Channel的就绪事件，注册Channel到WorkerGroup的Selector
  - WorkerGroup
    - 处理Selector上的读写事件（SubReactor）
    - 将其转发到其ChannelPipeline(业务处理链)中处理。（业务线程池）
- <img src="0JavaSummary.assets/image-20210726081729271.png" alt="image-20210726081729271" style="zoom: 50%;" />



详细版本：

![img](0JavaSummary.assets/47f4427f8820af163ca9cd1f545bf2c9.jpg)

- 最佳实践

1. 创建两个 NioEventLoopGroup 隔离 NIO Acceptor 和 NIO I/O
2. 尽量不在 ChannelHandler 中启动用户线程（用户线程是指的是在Reactor模式之外的业务线程）
3. 解码要放在 NIO 线程调用的解码 Handler 中进行，不要切换到用户线程中
4. 如果业务逻辑简单，没有阻塞、数据库操作、网络操作等，直接在 NIO 线程上完成业务逻辑而不要切换到用户线程
5. 如果业务逻辑复杂，则尽快释放 NIO 线程，交由用户业务线程处理

### Netty零拷贝

- mmap+write， 接收和发送`ByteBuffer`采用`DIRECT BUFFERS`，使用堆外直接内存进行`Socket`读写。**(减少用户态和内核态的对象拷贝)**

- 组合和拆分Buffer:  CompositChannelBuffer·对象，可以将多个ByteBuf 合并为一个逻辑上的 ByteBuf, 避免了各个 ByteBuf 之间的拷贝**（减少在用户态中，对象与对象的拷贝）**

- 文件传输采用了FileChannel的transferTo方法，直接将文件缓冲区的数据发送到目标Channel **(减少用户态和内核态的对象拷贝)**

Netty 通过提供的 Composite（组合）和 Slice（拆分）单个传输的报文，两种 Buffer 来实现零拷贝。看下面一张图会比较清晰：
![img](0JavaSummary.assets/20200226205251960-1624866709357.png)

```java
 class CompositeChannelBuffer extends AbstractChannelBuffer {
 
    private final ByteOrder order;
    private ChannelBuffer[] components; // 用来保存的就是所有接收到的 Buffer
    private int[] indices; // Indices 记录每个 buffer 的起始位置
    private int lastAccessedComponentId; // 记录上一次访问的 ComponentId
    private final boolean gathering;
 
    public byte getByte(int index) {
        int componentId = componentId(index);
        return components[componentId].getByte(index - indices[componentId]);
}
```

- 内存池 ：ByteBufAllocator 用于分配 ByteBuf，使用了池化技术
  - 原因：缓冲区Buffer是堆外内存，回收耗时
  - 方案：基于内存池的缓冲区重用机制。
    - 实现是PooledByteBufAllocator，
    - 最底层分配直接内存是Java的`ByteBuffer.allocateDirect`

  - 使用堆外内存的条件：
    - 要有cleaner方法去释放(本质是软引用，Soft Reference)
    - io.netty.noPreferDirect = false

> Kafka的RequestChannel的MemoryPool作用是什么，怎么分配内存？和Netty有区别吗？有，Kafka是堆内分配的。

### 常见的问题

1. Netty 是什么？Netty 的特点是什么？

   - 高性能、**异步事件驱动的NIO**框架，它提供了对TCP、UDP和文件传输的支持。

2. Netty 的优势有哪些？为什么要用  Netty？Netty 的应用场景有哪些？

   - 线程模型Reactor可灵活配置
   - 自带编解码器解决 TCP 粘包/拆包问题。
   - 比直接使用 Java 核心 API 有更高的吞吐量、更低的延迟、更低的资源消耗和更少的内存复制。
   - 成熟稳定，大型项目考验，比如 Dubbo、RocketMQ 等等。

   Netty 主要用来做**网络通信** :

   - **RPC 框架**
   -  **HTTP 服务器**
   - **即时通讯系统**，**消息推送系统** 

3. BIO、NIO和AIO的区别？NIO的组成？

   - BIO 同步阻塞
   - NIO 异步阻塞，IO多路复用模型
   - AIO 异步非阻塞
   - NIO：Buffer、Channel、Selector组成

4. Netty的线程模型？Netty 核心组件有哪些？分别有什么作用？

   - Reactor模型
     - Selector：**基于Selector对象实现I/O多路复用，监听多个连接的Channel事件**
     - Channel: 执行网络I/O操作。**EventLoop** 负责处理注册到其上的**Channel** 处理 I/O 操作，两者配合参与 I/O 操作。
     - EventLoop :负责监听网络事件并调用事件处理器进行相关 I/O 操作的处理。
   - ChannelFuture：封装请求的返回结果
   - ChannelHandler 和 ChannelPipeline
     - **ChannelHandler** 是消息的具体处理器，处理读写操作、客户端连接。
     - ChannelPipeline 为 ChannelHandler 的链，定义了用于沿着链传播inBound和OutBound事件流的 API 。

5. EventloopGroup 了解么?和 EventLoop 啥关系? Bootstrap 和 ServerBootstrap 了解么？

- NioEventLoopGroup：管理EventLoop的生命周期，线程池。
- (NioEventLoop)：处理多个Channel上的事件，线程。
- Bootstrap、ServerBootstrap
  Netty应用通常由Bootstrap开始，配置整个Netty程序，串联各个组件，
  - Bootstrap类是客户端程序的启动引导类
  - ServerBootstrap是服务端启动引导类

![EventLoop and EventLoopGroup 6. Netty source code analysis of the - Code  World](0JavaSummary.assets/1739214-20190922162938492-493061362-1623906811385.png)

1. NIOEventLoopGroup源码？NioEventLoopGroup 默认的构造函数会起多少线程？

   - MultithreadEventLoopGroup -> MultithreadEventExecutorGroup-> AbstractEventExecutorGroup

   ```java
       // 从1, 系统属性，CPU核心数*2 这三个值中取出一个最大的
       //可以得出 DEFAULT_EVENT_LOOP_THREADS 的值为CPU核心数*2
       private static final int DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
   
       // 被调用的父类构造函数，NioEventLoopGroup 默认的构造函数会起多少线程的秘密所在
       // 当指定的线程数nThreads为0时，使用默认的线程数DEFAULT_EVENT_LOOP_THREADS
       protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
           super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
       }
   ```

   

2. Netty 服务端和客户端的启动过程了解么？默认情况  Netty 起多少线程？何时启动？

```java
 // server 端启动过程
    void startServer(int port) {
        // 1.bossGroup 用于接收连接，workerGroup 用于具体的处理
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //2.创建服务端启动引导类：ServerBootstrap
            ServerBootstrap b = new ServerBootstrap();
            //3.给引导类配置两大线程组,确定线程模型
            b.group(bossGroup, workerGroup)
                    // (非必备)打印日志
                    .handler(new LoggingHandler(LogLevel.INFO))
                    // 4.指定 IO 模型： 通过channel()方法给引导类 ServerBootstrap指定了 IO 模型为NIO
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ChannelPipeline p = ch.pipeline();
                            //5.可以自定义客户端消息的业务处理逻辑
                            p.addLast(new MyRegistryHandler());
                        }
                    });
            // 6.bind端口,调用 sync 方法保证bind完成。
            ChannelFuture f = b.bind(port).sync();
            // 7.阻塞等待，直到服务器Channel关闭 (closeFuture()方法获取Channel 的CloseFuture对象,然后调用sync()方法)
            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            //8.优雅关闭相关线程组资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    // client 端启动过程
    void startClient(String host, int port) {
        //1.创建一个 NioEventLoopGroup 对象实例
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            //2.创建客户端启动引导类：Bootstrap
            Bootstrap b = new Bootstrap();
            //3.指定线程组
            b.group(group)
                    //4.指定 IO 模型
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline p = ch.pipeline();
                            // 5.通过 .handler()给引导类创建一个ChannelInitializer ，然后制定了客户端消息的业务处理逻辑 RpcProxyHandler 对象
                            p.addLast(new RpcProxyHandler());
                        }
                    });
            // 6.尝试建立连接。 通过 addListener 方法可以监听到连接是否成功，打印出连接信息。
            ChannelFuture f = b.connect(host, port).addListener(future -> {
                if (future.isSuccess()) {
                    System.out.println("连接成功!");
                } else {
                    System.err.println("连接失败!");
                }
            }).sync();
            // 7.等待连接关闭（阻塞，直到Channel关闭）
            f.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
```

1. Netty 发送消息有几种方式？

2. TCP 粘包/拆包的原因及解决方法？(OLS sina是怎么处理的，杰哥的RedisEncoder是怎么写的)

   **本质上TCP是流式协议，消息无边界**

   - 粘包原因：
     - 发送方写入的数据 小于 Socket缓冲区
     - 接受方接受数据不及时
   - 半包原因
     - 发送方写入的数据 大于 Socket缓冲区
     - 发送的数据大于MTU，必须拆包

   - 解决办法：
     - 封装成帧，固定长度字段存内容的长度信息，每次先解析消息有多长，然后读取后续的内容。（Netty的实现：LengthFieldBasedFrameDecoder, LengthFieldPrepender）

   **1.使用 Netty 自带的解码器**（一次编解码：解决粘包和半包问题，byteBuffer--> byteBuffer）

   - **LineBasedFrameDecoder** : 发送端发送数据包的时候，每个数据包之间以换行符作为分隔，LineBasedFrameDecoder 的工作原理是它依次遍历 ByteBuf 中的可读字节，判断是否有换行符，然后进行相应的截取。
   - **DelimiterBasedFrameDecoder** : 可以自定义分隔符解码器，**LineBasedFrameDecoder** 实际上是一种特殊的 DelimiterBasedFrameDecoder 解码器。
   - **FixedLengthFrameDecoder**: 固定长度解码器，它能够按照指定的长度对消息进行相应的拆包。
   - **LengthFieldBasedFrameDecoder**：最推荐++。

   **2.自定义序列化编解码器**(二次编解码：解析byteBuffer-> Java Object)

   * MessageToMessageDecoder Netty 自带的

   - RedisDecoder\RedisEncoder：
   - OLS 使用的是LengthFieldBaseFrameDecoder

3. 了解哪几种序列化协议？如何选择序列化协议？从什么角度选择序列化协议？（TODO)

   - 专门针对 Java 语言的：Kryo，FST 等等

   - 跨语言的：Protostuff（基于 protobuf 发展而来），ProtoBuf，Thrift，Avro，MsgPack 等等

4. Netty 支持哪些心跳类型设置？Netty 长连接、心跳机制了解么？

   - 在 TCP 保持长连接的过程中，可能会出现断网等网络异常出现，异常发生的时候， client 与 server 之间如果没有交互的话，他们是无法发现对方已经掉线的。为了解决这个问题, 我们就需要引入 **心跳机制** 。
   - **心跳机制的工作原理**是: 在 client 与 server 之间在一定时间内没有数据交互时, 即处于 idle 状态时, 客户端或服务器就会发送一个特殊的数据包给对方, 当接收方收到这个数据报文后, 也立即发送一个特殊的数据报文, 回应发送方, 此即一个 PING-PONG 交互。所以, 当某一端收到心跳消息后, 就知道了对方仍然在线, 这就确保 TCP 连接的有效性.
   -  Netty 层面通过编码实现。通过 Netty 实现心跳机制的话，核心类是 IdleStateHandler 。（为什么不直接用TCP：SO_KEEPALIVE？不灵活，不容易控制，因此常常在应用层自己实现）

## 6 kafka

分布式流处理框架，企业级的消息引擎

- 缓冲和削峰
- 解耦和扩展性
- 冗余
- 异步通信

### 概念

- LEO、LSO、AR、ISR、HW
  - LEO:Log End Offset。日志末端位移
  - LSO:Log Stable Offset，事务
  - AR：Assigned Replicas。所有副本集合
  - **ISR**:In-Sync Replicas，与 Leader 同步的副本
    - 副本是否 ISR？replica.lag(10s)
  - **HW**：高水位值（High watermark）,消费者可见，ISR中最小的LEO。
    - HW作用：HW和LEO共同完成副本同步

<img src="0JavaSummary.assets/image-20210723144505788.png" alt="image-20210723144505788" style="zoom:50%;" />

### 日志格式

- 文件格式和Request消息一致，因此可以使用ZeroCopy技术直接存储在磁盘上

- v0

  <img src="0JavaSummary.assets/120302.png" alt="img" style="zoom:80%;" />

- v1

  <img src="0JavaSummary.assets/120304.png" alt="img" style="zoom:67%;" />

- v2

  <img src="0JavaSummary.assets/120306.png" alt="img" style="zoom:80%;" />

- DumpLogSegment工具，可以查看片段内容，显示消息的偏移量、校验和、魔术、消息大小和压缩算法
- LogSegment有Index（基于mmap实现），把偏移量映射到片段文件和偏移量在文件里的位置；kafka不维护index的校验和，损坏则重新读取消息生成index
- Broker Map结构segments维护着当前的LogSegment的引用，LogSegment包含日志和index，fetch请求先在map中找到对应的LogSegment，接着读取出FetchDataInfo（Partition.read() -> Log.read() -> LogSement.read() -> LogSegment.translateOffset()）

### 副本

- 作用：冗余（无横向扩展、无数据局部性访问特性）

- Leader 和 Follower 区别

  - Leader读写
  - Follower PULL同步数据（2.4 ，可读）

- ISR(In Sync Replica)

  - 保持与Leader同步的副本。（lag=10s）

  1. ACK=all，ISR数据同步，回复ack
  2. ACK=all，只有当ISR的大小大于最小的ISR集合，才能写成功。（**一致性和可用性的折衷，交给用户来决定**）

- **ISR收缩和扩容、如何管理**

  - 判断标准：lag值，`ReplicaManager`启动2个定时任务
    - 收缩maybeShrinkIsr：lag值大于10s
    - 扩容maybeExpandIsr：follower的LEO >= current HW
  - isr-expiration检测每个分区是否需要缩减ISR集合
    - 将变更后的数据记录到ZooKerper`/brokers/topics/partition/state`节点
    - 更新缓存isrChangeSet
    - 尝试更新HW
  - isr-change-propagation 检查isrChangeSet，创建ISR变更通知事件
    - 将变更后的数据记录到ZooKerper`/isr_change_notification/isr_change_sequence_number`节点
    - 清空缓存isrChangeSet
      - ISR变更事件创建条件
        - 距离上一次ISR集合变化超过5s
        - 上一次写入ZK超过60s
    - Controller Watch这个节点，获取元数据更新信息，处理后，删除顺序节点

- Leader 选举

  - 思想：从 AR 中挑选首个在 ISR 中的副本，作为新 Leader
  - 是否开启unclean选举
  - 一种场景，一种选举策略。
    - OfflinePartition: 分区上下线
    - ReassignPartition：手动kafka-reassign-partitions 
    - PreferredReplicaPartition ：手动kafka-preferred-replica-election
    - ControlledShutdownPartition ：Broker 正常关闭

**副本同步全流程，即HW和LEO是如何被更新的？**

- Leader和Follower副本的HW和LEO存储在哪里

<img src="0JavaSummary.assets/image-20210723144936923.png" alt="image-20210723144936923" style="zoom:50%;" />

- Leader 处理生产者请求

  1. 写到本地磁盘，更新自己的LEO。
  2. 更新分区高水位值。
     i. 获取 本地远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。
     ii. 获取 Leader 副本高水位值：currentHW。
     iii. 取他们的最小值

- Leader处理Follower Fetch 请求

  1. 读取磁盘（或页缓存）消息

  2. 更新远程副本 LEO 值，从Follower来的请求的LEO值

  3. 更新分区高水位值

     i. 获取 本地远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。
     ii. 获取 Leader 副本高水位值：currentHW。
     iii. 取他们的最小值

- Follower 拉取 Leader 消息

  1. 写到本地磁盘，更新自己的LEO
  2. 更新高水位值。
     i. 获取 Leader 发送的高水位值：currentHW。
     iii. 更新高水位为 `min(Leader的currentHW, 自己的currentLEO)`

- 举例

  - 初始状态都是0，生产者发起请求，Follower发起Fetch请求

    ​	<img src="0JavaSummary.assets/image-20210725234557899.png" alt="image-20210725234557899" style="zoom:40%;" />


**Follower拉取leader消息的源码表述**

- AbstractFetcherThread 线程的 doWork ，入口方法
  - 日志截断（truncate）+ 日志获取（buildFetch）+ 日志处理（processPartitionData）
  - truncate 方法：根据 Leader 副本返回的位移值和 Epoch 值执行本地日志的截断操作。
  - buildFetch 方法：为一组特定分区构建 FetchRequest 对象所需的数据结构。
  - processPartitionData 方法：处理从 Leader 副本获取到的消息，主要是写入到本地日志中。
- 子类 ReplicaFetcherThread 类
  - Follower 副本利用 ReplicaFetcherThread 线程实时地从 Leader 副本拉取消息并写入到本地日志，
    从而实现了与 Leader 副本之间的同步。

### Leader Epoch

Leader 和 Follower 的消息序列在实际场景中不一致，如何确保一致性

- 高水位机制缺陷（无法保证 Leader 连续变更场景下的数据一致性）

  <img src="0JavaSummary.assets/image-20210725235931924.png" alt="image-20210725235931924" style="zoom: 25%;" />

  - 日志丢失场景：前提是**Broker 端参数 min.insync.replicas 设置为 1**， 2台Broker同时宕机，低水位的Broker B先启动起来，成为leader，当Broker A恢复，发现现在的HW是1，截断日志。

    - B 重启回来后，需要向 A 获取 Leader 的 LEO 值=2
    - A的LEO值比B大，缓存中也没有比2大的，不截断日志
    - 当 A 重启回来后，执行与 B 相同的逻辑判断，发现也不用执行日志截断
    - Producer向 B 写入新消息时，副本 B 所在的 Broker 缓存中，会生成新的 Leader Epoch 条目：[Epoch=1, Offset=2]

    <img src="0JavaSummary.assets/image-20210725235403829.png" alt="image-20210725235403829" style="zoom: 50%;" />

  - 日志不一致场景：前提是一样的，2台Broker同时宕机，Broker A的HW = 2， Broker B的HW =1 ,Broker B先启动起来，成为leader，接受了一条生产消息，HW==> 2；Broker A活过来，HW和leader的HW是一样的，不拉取消息。

- 引入Leader Epoch 机制

> Leader 和 Follower 的 HW 值更新时间是存在错配的，Follower 的 HW 更新永远落后于 Leader 的 HW。造成“数据丢失”或“数据不一致”

- 是什么？Leader Epoch是一种机制，一种概念。分为2个部分
  - Epoch。一个单调增加的版本号。Leader变更，版本号增加。

  - 起始位移（Start Offset）。Leader 副本在该 Epoch 值上写入的首条消息的位移。
  - **每个分区都缓存 Leader Epoch 数据**，定期持久化到checkpoint 文件
- 如何解决？

  - 每次活过来的follower去Leader拉取Leader的LEO值，以这个值来作为判断是否做同步的标准。


- 典型的应用场景：
  - 替换高水位值在日志截断中的作用。
  - 当分区存在 Leader Epoch 值时，将副本的本地日志截断到 Leader Epoch 对应的最新位移值处，` truncateToEpochEndOffsets`
  - 如果分区不存在对应的 Leader Epoch 记录，使用原来的高水位机制，将日志调整到高水位值处。`truncateToHighWatermark`

### 无消息丢失

- Broker
	- 设置 unclean.leader.election.enable = false
	- 设置 replication.factor >= 3。目前防止消息丢失的主要机制就是冗余。
	- 设置 min.insync.replicas > 1，如果生产时isr不满足最小同步副本数，则收到异常NotEnoughReplicasException
	- replication.factor = min.insync.replicas + 1，可用性和一致性的权衡。
	- 单个broker对分区个数有限制，分区越多，占用内存越多，完成leader选举所需时间也越长
	- 极端情况，Kafka生产者写消息不丢失，page cache 改成同步落磁盘
	- 硬件需求
	   1. 磁盘速度决定producer的延迟性能
	   2. 磁盘容量决定数据冗余量以及存储周期
	   3. 内存可用于做页面缓存供kafka缓存正在使用中的日志片段
	   4. 网络决定最大吞吐量
	   5. kafka对cpu要求不高，但是cpu会影响加解压缩以及GC停顿
	
- Producer
  - 设置 acks = all
  - 使用 producer.send(msg, callback)，用 Callback 来处理异常情况
  - 设置 retries、retry.backoff.ms，重试几次，每次重试的间隔
  - buffer.memory、block.on.buffer.full/max.block.ms：内存缓冲的大小的，默认值32MB
    - buffer.memory设太小，消息写入内存缓冲，但Sender线程发送不及时，被写满，阻塞用户Producer线程（压测）
  - batch.size，多条消息合成一个批次发往同一分区时，批次占用内存的大小。默认值16KB
    - 提升batch.size，提升吞吐，延迟高
  - linger.ms，一批次最多等待时间，50ms。
  - client.id
  - max.in.flight.requests.per.connetion，producer 收到服务器响应之前可以发送多少个消息，影响吞吐量、占用内存、消息顺序性
  - request.timeout.ms（请求超时时间）、metadata.fetch.timeout.ms（元数据请求超时时间）
  - max.block.ms 最大等待时间，比如send()等待元数据信息返回
  - max.request.size，请求消息的最大大小

  - produce的重试设置
    - broker返回的错误有两种，一种可重试解决，例如 LEADER_NOT_AVAILABLE
    - 另一种不可重试，比如网络中断，重试有可能导致重复消息
    - Kafka 无法避免消息重复，在应用程序中加入唯一标识符检测重复，业务“幂等”

- Consumer端

  - 配置说明

    - auto.offset.reset（seekToBegining()，seekToEnd()）
    - enable.auto.commit、auto.commit.interval.ms 自动提交位移
    - partition.assignment.strategy，Range、RoundRobin
    - client.id
    - max.poll.records，单次poll的数据条数
    - fetch.min.bytes，消费者从服务器获取记录的最小字节数，borker 会等到有足够的可用数据时才把数据返回给消费者
    - [fetch.max.wait.ms](http://fetch.max.wait.ms)，消费者等待 broker 返回的最长时间，默认 500ms
    - max.partition.fetch.bytes，每个分区返回给消费者最大的字节数，默认 1MB
    - [session.timeout.ms](http://session.timeout.ms)，消费者与服务器断开连接的判断时间，默认 3s；若 consumer 没有在此时间内发送心跳给 GroupCoordinator，则被认为死亡；一般将 [heartbeat.interval.ms](http://heartbeat.interval.ms) 配置为 session.timeout.ms的 1/3

  - offset提交

    - 消费完，提交offset，设计时需要考虑到Rebalance：调用 subscribe() 时传入 ConsumerRebalanceListener
    - 当部分消息处理失败时的重试，有两种模式
      1. 可提交最后一个成功处理的偏移量，把未处理的消息保存到缓冲区，调用消费者pause()方法暂停轮询返回的数据，保持轮询的同时尝试重新处理，成功或者达到重试次数上限（记录错误丢弃消息），然后调用rersume()方法恢复消费者轮询数据
      2. 可将错误写入单独的 topic，然后继续，再由其他 consumer 消费该 topic 单独处理

    - 想把 offset 保存到别的数据库里，可使用 seek() 和 ConsumerRebalanceListener 配合

    


1. Producer丢失消息，从哪里分析？什么原因会导致消息丢失？解决？

   - 没有发送到Broker，由于网络抖动 
   - 消息太大了，Broker不接收

2. Consumer端丢失数据的现象，想读的消息没有读上？场景，解决？

   - Consumer获取消息，开启了多个线程处理，而**自动地向前更新了Offset**。如果某个线程运行失败，Consumer就丢失了消息。
   - (有一个方案是Consumer不要开启自动提交offset，让所有线程消费完才手动提交。**会不会出现消息被消费了多次的情况呢?有没有好的方案**)

   >  min.insync.replicas=1理解？
   >
   >  思考题：Kafka有一个隐私的消息丢失场景：增加主题分区。当增加主题分区后，如果Producer先于Consumer感知到这个分区，而Consumer设置的是从latest的地方读取数据，那么就会存在数据丢失。有什么解决办法么？


### Producer

* TCP连接是如何建立的

  * Producer会与`bootstrap.servers`创建TCP连接，不用的，超时9分钟自动关闭。（大集群有优化空间：3-4台配置足矣）
  * 创建KafkaProducer之后，后台会启动Sender线程，该线程运行时首先会创建与Broker的连接。
  * KafkaProducer 向某一台Broker再次请求集群的metadata，包括自己订阅Topic的所有meatadata
  * KafkaProducer向Topic 分区的leader的broker发起TCP连接，准备写数据。

  > KafkaProducer是线程安全的么？KafkaProducer和Sender线程共享只有RecordAccumulator类，使用ConcurrentMap<TopicPartiion,Deque>,TopicPartition是不可变类，Deque用到的地方，都加了锁。所以是线程安全的。

* TCP的连接除了初始化创建，可能还有什么？

  * Producer更新元数据
  * Producer消息发送时

- 幂等性如何实现？0.11，at least once + 幂等 = exactly once

  - 单分区不重复，单会话不重复。(重启，丢失缓存)

  - **实现**：解决单会话ACK 超时导致重复
    - Broker缓存消息，根据PID和SequenceNumber判重
    - ProducerID：唯一的ProducerID，标识client
    - SequenceNumber：TopicPartition级别，每条消息带着，Broker判重。

  <img src="0JavaSummary.assets/image-20210713143421514.png" alt="image-20210713143421514" style="zoom:50%;" />

  * 申请PID
    * Client：InitProducerIdRequest 发送给连接数最少的Broker
    * Broker: TransactionCoordinator的ProducerIdManager生产id，TransactionCoordinator负责与Producer通信，更新message的事务状态。
    * PID 申请是向 ZooKeeper 申请，类似于CompareAndSwap的方式，来写入PID，写入成功，就申请成功；失败就重试。

- Producer请求过程

<img src="0JavaSummary.assets/kafka-idemoptent.png" alt="Producer 幂等性时处理流程" style="zoom: 50%;" />

1. KafkaProducer 的 `send()` 将数据添加到 RecordAccumulator
2. Producer 发送线程 Sender申请PID,`sendProducerData()` 方法发送数据
3. 进入Broker端逻辑，判重在Broker端，batchMetadata 缓存batch ，5（社区测试）

- 什么情况不能保证补充不丢不重
  - 有 topic-partition 的 batch 重试多次失败，超时被移除，这sequence number 无法连续，需要重置ProducerID，Broker端缓存被清空。

* 如何实现写消息一定有序
  * Producer设置，影响Broker，
  * MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION =1，有序、性能
  * 当出现重试时，max-in-flight-request 可以动态减少到 1，在正常情况下还是按 5 （CA取舍)



- 事务性Producer：

  * 多分区不重复。

  * TransactionManager 实例，它的作用有以下几个部分：
    1. 记录本地的事务状态（事务性时必须）；
    2. 记录一些状态信息以保证幂等性，比如：每个 topic-partition 对应的下一个 sequence numbers 和 last acked batch（最近一个已经确认的 batch）的最大的 sequence number 等；
    3. 记录 ProducerIdAndEpoch 信息（PID 信息）。

#### 分区策略

实现负载均衡，高伸缩性，高吞吐量。

**如何自定义分区策略有哪些？**

编写一个具体的类实现`org.apache.kafka.clients.producer.Partitioner`接口

```java
int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
```

- 轮询策略：Producer API默认分区策略。很优秀，很常见。

- 随机策略：

- 消息键保序：Key-ordering，Kafka运行为每条消息定义key。在kafka不支持时间戳的时候，这个key放时间戳；后来，这个key可以保证某些消息进入到同一个分区，这个相当重要，有很多用途。

  ```java
  List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
  return Math.abs(key.hashCode()) % partitions.size();
  ```

  > Kafka 默认情况下，Producer没有指定key，就用轮询策略；如果指定了，就用key-ordering.

- 黏性分区器（Sticky Partitioner）是选择单个分区发送所有无Key的消息。一旦这个分区的batch已满或处于“已完成”状态，黏性分区器会随机地选择另一个分区并会尽可能地坚持使用该分区——像黏住这个分区一样

- 在大规模集群中，还有一种基于地理位置的分区策略。

### Consumer

- Consumer的TCP连接
  - 什么时候创建？调用KafkaConsumer.poll方法
  - 讲一下创建的过程？3次请求
    - 寻找FindCoordinator和获取集群的metadata（Consumer发起）-- 最后被关闭
    - 连接Coordinator（Consumer发起）-- 一直被复用
    - 连接分区副本的leader（Consumer发起）---一直被复用
    - TCP生命周期，默认9分钟
- ConsumerGroup是什么
  - 官网，可扩展、容错性的消费者机制
  - 多个Consumer实例。订阅主题，共同消费。某个挂掉，rebalance。
- offset
  - 每个消息在partition的唯一ID
- __consumer_offsets
  - 注册消费者以及保存位移值，GroupCoordinator管理、读写
- 源码如何设计：采用单线程来获取消息
  - 双线程
    - poll设计负责获取消息
      - 所有逻辑都在poll的大循环中实现，包括获取Metadata、连接GroupCoordinator、发送Fetch请求、提交Offset、发送心跳，这些操作都先加入队列，之后才在poll中发送出去，其中心跳请求会加入DelayedQueue中定期出队
      - 新版本的Kafka心跳采用了单独的线程
    - 心跳线程。规避因消息处理速度慢而下线，引发rebalance。
  - 异步非阻塞，适合流式
- 保证消费的时序性？
  - 难以保证的原因：
    - Producer是多台机器，没有统一的分布式时钟
    - Consumer是多台机器，无法保证不同Consumer的消费顺序
    - 消息重传
    - Topic Partition是多分区
  - 全局有序和局部有序：单一分区有序
    - Kafka的Producer "max.in.flight.requests.per.connection=1"
    - hash分发到同一个分区
  - 业务保证顺序的方法
    - 以Producer、Consumer端发送时间戳为准
    - 发送消息时，采用唯一自增ID
    - 缓存时间戳，发送时，给缓存放入时间戳；消费时，去缓存查询是否是最新的
- 不重复消费的方法
  - Kafka端，保证生产是幂等的，消费也开启幂等
  - 业务端，幂等性设计
    - 全局分布式ID，消费完，就放入到缓存，代表数据已经被消费
    - 数据库去重，比如订单ID和时间戳作为索引
- 监控Consumer消费进度
  - Consumer Lag: 消费者落后生产者的条数，理想情况要等于0
    - 如果太大，要消费的数据就不在页缓存，丢失Zero-Copy
  - Lead：records-lag-max 和 records-lead-min
    - 最新消费消息的Offset与分区最老的Offset差值
    - 如果接近0，意味着一直在消息最老的消息，丢失消息

### Coordinator

- Broker有Coordinator组件，负责协调ConsumerGroup的消费情况

- Consumer的两种请求

  - JoinGroup：当组内成员加入组时，向Coordinator发送，上报订阅的TopicPartition
  - Coordinator把所有Consumer订阅信息通过 JoinGroup Response，然后发给领导者，由领导者统一做出分配方案后
  - SyncGroup：领导者向Coordinator发送 SyncGroup 请求，包含分配方案

- 新成员2加入

  - 寻找GroupCoordinator
    - ConsumerCoordinator调用ensureCoordinatorReady()获得group的地址（首先调用leastLoadedNode()寻找连接最少的broker，向其发送请求寻找groupCoordinator）
  - 发送JoinGroup
    - client端发送JoinGroup请求，若group不存在则创建新的group，状态置为Empty
  - Leader选举以及等待
    - 第一个加入的member被选为consumer leader
    - GroupCoordinator状态置为PreparingRebalance
    - 接着会等待一定时间，等待预期的consumer陆续提交JoingGroup后，group进入CompletingRebalance状态
    - GroupCoordinator给client返回封装有所有member的response
  - 分配方案
    - leader收到JoinGroup的response后，生成assignment
    - client端向coordinator发起SyncGroup请求，若client端是leader则在sync请求中提交分配方案，follower发送的则是一份空的列表
    - coordinator收到leader的请求后，将分配方案作为SyncGroup的response分发给follower，group状态转移为Stable

  ![image-20210727203145134](0JavaSummary.assets/image-20210727203145134.png)

- 组成员崩溃离组，session.timeout.ms 

  ![image-20210727203353716](0JavaSummary.assets/image-20210727203353716.png)

- Rebalance，Coordinator对组内成员提交位移的处理

  ![image-20210727203506667](0JavaSummary.assets/image-20210727203506667.png)

- 参数

  - rebalance.timeout.ms，worker在rebalance后加入group的最大时间
  - group.initial.rebalance.delay.ms，空Group接收到第一个JoinGroup后延迟多久后才开始Rebalance
  - session.timeout.ms，心跳线程超时时间
  - max.poll.interval.ms，执行线程超时时间
  - Static member、Incremental Rebalance
    - Reduce unnecessary downtime due to unnecessary partition migration: i.e. partitions being revoked and re-assigned.
    - Better rebalance behavior for falling out members.

- GroupState

  1. Empty
     - 没有任何member的group，一直保留直至所有的offsets都过期失效（offsets都定期清除后，状态转移为Dead）
     - 这个状态也可适于仅提交offset而没有member的用法
     - 只对JoinGroup请求有正常的响应，其他均返回错误
     - 新member发起JoinGroup请求则转移至PreparingRebalance
       - 当group被移除，状态转移至Dead
  2. PreparingRebalance
     - 对心跳请求、Sync请求返回REBALANCE_IN_PROGRESS，移除member离开group的请求，暂停新的或已存在的member发送的JoinGroup请求，直到所有预期的member都已加入
     - 当等待时间结束前members完成Joined，状态转移至CompletingRebalance
       - 所有members都离开了group，状态转移至Empty
       - 当group被移除，状态转移至Dead
  3. CompletingRebalance/AwaitingSync
     - Group正在等待leader的分配方案，暂停follower的SyncGroup请求直到状态变成Stable
  4. Stable
     - 正常回复心跳请求，以当前的分配方案回复SyncGroup请求，若当前client端与coordinator的metadata匹配则以当前group metadata回复client端的JoinGroup请求
     - member心跳异常、member离组、leader发送JoinGroup请求、follower以新的metada发起JoinGroup请求则进入PreparingRebalance状态
     - 分区迁移导致group移除则进入Dead状态
  5. **Dead**：group的最终状态，没有状态转移

![image-20210727202239494](0JavaSummary.assets/image-20210727202239494.png)

- 一个消费者组最开始是 Empty ，开始Rebalance后，处于 PreparingRebalance 状态等待成员加入，之后变更到CompletingRebalance 状态等待分配方案，最后到 Stable 状态完成。
- 当有新成员加入或已有成员退出时，消费者组的状态从 Stable 直接跳到PreparingRebalance 状态，此时，所有现存成员就必须重新申请加入组。当所有成员都退出组后，消费者组状态变更为 Empty。



> 项目经验来了：log cleaner 线程挂掉，导致消费端出现`Marking Coordinator Dead!` 
>
> 原因是：log cleaner线程挂掉---> offset文件越来越多--> broker 内存维护了offsetMap，这Map越来越大，导致offsetMap无法在添加数据---> 导致broker不承认自己是coordinator。 ---> 而消费者找Coordinator的时候，又找到这个broker。---> 导致这个consumer就无法消费任何数据，出现上面的错误。



### Rebalance

- 坏处
  
  - STW，Consumer 端 TPS
  - 慢
  - 无局部性原理
  
- 原因
  - Consumer数量
  - 订阅Topic数量
  - 订阅Topic Partition数
  
- 策略，由Coordinator定
  - 轮询
  - StickyAssignor，粘性策略，尽可能地保留之前的分配方案，尽量达到分区分配的最小变动

#### 如何避免计划外Rebalance？

主要是解决原因第1条

> Coordinator认为Consumer实例挂了？
>
> 1. 心跳，`session.timeout.ms=10s`
> 2. 心跳时间间隔，`heartbeat.interval.ms`
> 3. 拉取间隔，`max.poll.interval.ms=5分钟`（一次消费一批消息的时间如果超过5分钟，Consumer就会主动要求离开）
> 4. Consumer端 GC 参数设置不合理

针对这些1、2、3分别有哪些实战经验呢？

1. 规避第1类，Consumer未能及时发送心跳，导致Consumer被踢出Group而引发。 

   - `sesstion.timeout.ms = 6s`（尽快让不合格的Consumer，早日离开）
   - `heartbeat.interval.ms=2s`（保证Consumer在dead之前，至少发送了3次心跳请求。）
2. 规避第3类，消费时间过长，增大max.pool.interval.ms`。原则是给业务留时间。
3. Consumer端是否频繁出现Full GC导致长时间的STW。

* 如何确定Coordinator的broker
  * `partitionId=Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)`这个副本的leader

### Offset

- 自动提交：默认5s，Consumer后台启动1个线程提交位移。逻辑上来讲，poll先提交上一批次的offset，在拉取数据。
  - Rebalance会出现消费重复，5s提交，但是3s出现Rebalance
- 手动提交有3种
  - 同步commitSync() 直接阻塞，直到成功。
  - 异步commitAsync() 不阻塞，有callback。缺点是什么？遇到问题后，不能重试提交offset操作。因为重试其实也没有用，因为它自身的offset是过期的。
  - 更加精细化的同步和异步的

- 最佳实践：使用2者，利用commitSync的自动重试来避免由于网络瞬间抖动和Broker GC导致失败，兼顾用commitAsync来提升TPS。

- 如何解决CommitFailedException？

  - 原因：`KafkaConsumer.commitSync() `, Rebalance重新分配分区，Consumer提交offset到一个不再属于它消费的partition。

  - 解决：如果这个Consumer的心跳发的不及时，说明它本身不合格，让它退出就退出了，不管；另外，如果是一批消息消费加上业务处理时间大于了`max.poll.interval.ms=5分钟`，**这就是很经典的场景。而且真实存在。解决方法有4个方向**

        1. 减少下游处理单条消息的时间，优化业务系统，是最值得的事情
        2. 加大`max.poll.interval.ms`，原则就是要计算一下总时间= 平均时间 * 一批总条数（默认500）
        3. 降低一次poll的总条数：`max.poll.records`
        4. 最高级的，下游多线程处理加速。比如Flink的KafkaConsumerThread就是这样。

    > 思考：推荐第1个，其次2、3个，最后一个难，不容易处理offset的提交。
    >
    > 1. 发生这个异常，会终止这个Consumer继续消费吗？
    >
    > 2. 如何写一个多线程处理的高效Consumer？

### 多线程Consumer方案

consumer 是单线程。1个是消费主线程，1个是心跳线程。

两种线程方案：

1. 开启多个线程，每个线程里都有一个自己的kafka Consumer 实例，共享一个ConsumerGroup。一个线程的逻辑是：poll数据---> process数据--->再次poll数据
2. 分成①拉取数据线程和②处理业务线程 2个部分。拉取的数据直接交给另外一个线程池去处理。

| 方案  | 优点                            | 缺点                                                         |
| ----- | ------------------------------- | ------------------------------------------------------------ |
| 方案1 | 1. 实现简单                     | 1. 不容易扩展，最大的线程数不能超过partition的总数           |
|       | 2. 消息可以保证是有序地进行消费 | 2. 如果process耗时太多，容易发生Rebalance                    |
|       | 3. 速度快，没有线程间交互的开销 | 3. 占用资源，每个Consumer线程都需要维持TCP连接，还有暂用内存资源 |
| 方案2 | 1. 解耦了，方便扩展             | 1. 实现复杂                                                  |
|       |                                 | 2. 不能保证消费消息的有序性。让一个分区的数据让同一个线程消费，能够保证有序性。 |
|       |                                 | 3. 消费的链路拉长了，offset提交不好控制。（可以解决）        |



- 方案2如何提交offset？实现一套多线程+管理offset的方案
  - 约定成1个线程消费者，后面是线程池来进行业务处理

 #### 简单思路

- 消费线程：poll消息，扔给业务线程池。
- 当业务线程完成，提交offset
  - 业务线程异常，丢数据
  - Rebalance，重复消费

#### Partition粒度消费

让某个Partition只能被一个业务线程处理，处理完之后，在Consumer poll线程提交offset（不再是业务线程），再去拉取数据。

- 消费线程，也就是主入口线程：poll消息之后，进行3步骤

  - 按照Partition的粒度，分发到业务线程池，pause这些Partition的消费
  - 检查outstandingWorker，更新offset，resume已处理过的分区的下一次消费
  - 提交offset，有间隔的提交，最后最好清空一下offsetMap

- 线程池业务线程：一个线程只处理相同分区的数据

  - process 业务
  - 更新offset
  - 返回最新的offset、
  - 添加stop方法，方便rebance的时候，等待业务完成。

- Rebalance监听器：处理Rebalance的offset。

  - 一旦发生Rebalance，Consumer是停止了的，但是数据已经给了业务线程池了。

- 因此，Rebalance发生前，要给所有的业务线程，发送stop命令，停止处理；

  - 这个时候需要业务线程配合，要把最新的位移信息返回出来。
  - 然后Rebalance的监听器，提交这次位移。保证了数据的不丢不重复。
  - 最后，Rebalance开始，consumer 重新获得新的分区，开始从上一次提交的offset开始消费。完美的不丢不重复。

  > 整个方案的好处是：约定一个work任务只能处理同一个分区的数据，这个分区的数据不处理完，就不poll这个分区的数据。保证了两边的解耦，可以加大业务线程池来提高效率。（比如topic有20个partition，最多可以启动20个业务线程对它进行处理。相比于第1种方案，它也要启动20个consumer线程，但是存在Rebalance的风险。）
  >
  > - pause什么场景下用？pause是暂停一个分区拉取数据，而且不会触发Rebalance，常常`resume`搭配使用。
  > - wakeup什么时候用，解决什么问题的？wakeup是唤醒一个Consumer线程的，特别地用于abort 长时间阻塞的`poll`操作，可以用来停止一个Consumer。
  > - **Consumer在持续消费的时候，为什么poll总是能够准备地探测到下一次要拉取的信息？因为Consumer内部会维护一个指针，知道每次拉取了到了哪个位置，所以即使没commit offset，它也能够准备知道消费哪一条。但是重启Consumer或者Rebalance，这个指针就需要重置了。**



### Controller

- 做什么
  - 全局meta信息维护，管理Broker上下线、topic管理、管理分区副本分配、leader选举、管理所有副本状态机和分区状态机；通过zookeeper实现选举
  - Topic、Partition管理，Prefer 领导者选举：LeaderAndIsrRequest
  - Broker管理，元数据管理：UpdateMetadataRequest
  - StopReplicaRequest：使用场景：分区副本迁移和删除主题
- 是什么
  - 给 Broker 发送 3 类请求，即 LeaderAndIsrRequest、StopReplicaRequest 和 UpdateMetadataRequest，
- 脑裂，ActiveControllerCount>1，僵住
  - 背景：Controller FullGC太长，网络故障
  - 影响Topic的创建、修改、删除操作的**信息同步**。不影响现有topic的读写。
  - 解决：新的controller在zk生成新的controller epoch，并同步给broker，旧controller的指令，broker自动忽略。
- 元数据更新流程
  - Controller 启动，同步ZooKeeper
  - 异步发送给其他 Broker
  - 前面2步有时间差，导致Clients 访问的元数据不一定最新。（raft能解决吗）

### Broker流程

![image-20210702154239238](0JavaSummary.assets/image-20210702154239238-5211760.png)

- Client发送请求给SocketServer
- Acceptor收到请求，轮询的方式分配给Processor，创建TCP连接，放入到newConncetions队列
- Processor处理请求，把请求放入到RequestQueue
- KafkaRequestHandlerPool从请求队列中获取 Request 实例，然后交由 KafkaApis 的 handle 方法，执行真正的请求处理逻辑。
- KafkaRequestHandler 线程调用RequestChannel的SendResponse方法，将 Response 放入 Processor 线程的 Response Queue中
- 最后，Processor 线程发送 Response 给 Request 发送方

这张图更加易懂化。

<img src="0JavaSummary.assets/image-20210701092750874-5102873.png" alt="image-20210608215400269" style="zoom:80%;" />

- 1：Clients 或其他 Broker 发送请求给 Acceptor 线程

  - Acceptor 线程通过调用 accept 方法，创建对应的 SocketChannel，然后将该 Channel 实例传给 assignNewConnection 方法，等待 Processor 线程将该 Socket 连接 请求，放入到它维护的待处理连接队列中。后续 Processor 线程的 run 方法会不断地从该 队列中取出这些 Socket 连接请求，然后创建对应的 Socket 连接。
  - assignNewConnection 方法的主要作用是，将这个新建的 SocketChannel 对象存入 Processors 线程的 newConnections 队列中。之后，Processor 线程会不断轮询这个队列 中的待处理 Channel，并向这些 Channel 注册基于 Java NIO 的 Selector，用于真正的请求获取和响应发送 I/O 操作。

- 第 2 & 3 步：Processor 线程处理请求，并放入请求队列

  - 一旦 Processor 线程成功地向 SocketChannel 注册了 Selector，Clients 端或其他 Broker 端发送的请求就能通过该 SocketChannel 被获取到，具体的方法是 Processor 的 processCompleteReceives
  - Processor 线程处理请求，就是指它从底层 I/O 获取到发送数据，将其转换成 Request 对象实例，并最终添加到请求队列RequestQueue的过程。

- 4 步：I/O 线程处理请求

  - KafkaRequestHandler 线程循环地从请求队列中获取 Request 实例，然后交由 KafkaApis 的 handle 方法，执行真正的请求处理逻辑。

- 5 步：KafkaRequestHandler 线程将 Response 放入 Processor 线 程的 Response 队列

  - 这一步的工作由 KafkaApis 类完成。当然，这依然是由 KafkaRequestHandler 线程来完 成的。KafkaApis.scala 中有个 sendResponse 方法，将 Request 的处理结果 Response 发送出去。本质上，它就是调用了 RequestChannel 的 sendResponse 方法

- 6 步：Processor 线程发送 Response 给 Request 发送方

  - 最后一步是，Processor 线程取出 Response 队列中的 Response，返还给 Request 发送 方。具体代码位于 Processor 线程的 processNewResponses 方法
  - 最底层的部分是 sendResponse 方法来执行 Response 发送。该方法底 层使用 Selector 实现真正的发送逻辑。

### 移除Zookeeper

- 作用
  - 元数据管理、成员管理、Controller 选举。
  
- 为什么（2.8，KIP-500 移除zk）

  - 让 Kafka 独立
  - 性能问题，2百万的partitions，Controller切换，zk恢复2分钟，raft 32s
  - KIP-500 ：自研 Raft， Controller 自选举
  - raft暂时不支持ACLs

- 性能问题是如何解决的？Controller加载元数据为什么就快了？raft

  - Metadata as an Event Log 

    - 元数据作为 Log 储存
      - 有副本，高可用
      - 日志是顺序的
      - **增量同步**：Broker 间同步元数据，可增量同步。（速度快）
      - 可监控

    - Log 机制， Broker是 Consumer，从 Controller 拉取元数据，维护自己的消费offset。
    - 元数据 Topic 不能复用现有的副本机制，因为副本是由Controller管理的。（蛋生鸡）

  -  Controller quorum: 一组Controller组成Raft，只有一个Leader

    <img src="0JavaSummary.assets/image-20210722094249643.png" alt="image-20210722094249643" style="zoom:33%;" />

    - 与ZAB不同，Leader负责读写请求
    - 好处：切换Controller低延时，元数据可以缓存磁盘。



### 原理

####  高性能、高吞吐、低延时、速度快的原因

- 磁盘顺序读写

- Page Cache，避免Object消耗，避免GC

- 零拷贝

  - 生产者是`mmap+write `,写入到页缓存，页缓存映射文件
  - 消费者或者Follower: sendfile，数据从Page Cache 直接发送到网络

- Partition+LogSegment+二分查找索引，二进制格式文件: partition文件夹、LogSegment文件、多种索引

- 批量读写、批量压缩减少网络IO

  

#### Zero Copy

- Kafka: 生产者是`mmap+write`, 消费者或者Follow同步消息是`sendfile`
- 基于 mmap 的索引
- 日志文件读写TransportLayer， FileChannel 的 transferTo方法，操作系统的sendfile 
- 压缩和解压缩会丧失zero-copy的特性么？会
  - 写消息，解压缩校验
  - 读消息。可以sendfile

#### 一致性怎么保证的？不支持读写分离

* 避免不一致性
* 场景不适用，分离适用读负载很大
* 同步机制，Follower存在落后Leader的时间窗口，若Follower可读，须容忍消息滞后

#### 网络分区如何解决，分情况

* 单个 Broker 隔离

  * Controller自动移除它，一致性C保证

* Broker间不通

  * 副本备份出问题，ISR被收缩，一致性C保证

* 所有Broker 与 ZooKeeper 不通

  * Broker进入Zoobie，一致性bug，解决方法： **fencing**，比如 Leader Epoch

* **某个Broker 与 Controller 不通**

  * **元数据不一致**，无法感知到。因为Broker 是否活着完全是交由 ZooKeeper 。一旦某个 Broker 与 ZooKeeper 可通信，集群认为是正常的。（raft可以解决）
  * 解决方法：强制Controller重选举
  * 需考虑的问题：加载ZK的元数据很慢，200W的partition需要2分钟。
  * 怎么解决性能问题？参考：移除Zookeeper

  

### 调优

 * 目标不同，结果不同。吞吐量、延时、持久性和可用性
 * 优化 Kafka 的 TPS
     * Producer 端：增加 batch.size、linger.ms，启用压缩，关闭重试等。
     * Broker 端：增加 num.replica.fetchers，提升 Follower 同步 TPS，避免 Broker Full GC 等。
     * Consumer：增加 fetch.min.bytes 等

#### 1请求处理不及时

- Sina案例：RequestQueueTimeMs：3台broker高
  - RequestQueueTimeMs：Request 在 队列中的平均等候时间，单位是毫秒。等待时间过长，增加 I/O 线程的数量，加快队列的消费速度。
- 分析：热点topic，
- 解决：
  - 思路：
    - 应对突发流量，流量开始激增的时候，通过监控拿到这个指标，临时加大IO线程数。
  - 增加io线程数，加倍8，临时加大处理能力
  - 增加主题分区

> 

- EMC案例1：单分区2个副本的UUT主题， Broker A是 Leader，Broker B是follower。运行正常。后来新来了10台同类型的UUT，这10台UUT被同时调度在同一个时刻，进行测试，导致写日志流量激增。导致 Broker A 瞬间积压了大量的未处理 PRODUCE 请求。同事执行了 Preferred Leader 选举，将 Broker B 变成Leader。
  - 日志中出现了Broker A抛出的超时异常，Producer程序异常，失败。
- 分析：Producer的配置ack = all，Request TotalTimeMs： 3s
  - Leader/Follower 转换，未完成的 PRODUCE 请求会一直保存在 Broker A 上的 Purgatory 缓存，不断重试，超时异常，无法完成副本间同步。

- 解决Broker端：
  - 增加io线程数，临时加大处理能力
  - 增加主题分区

####   Producer 程序发送消息延时高

- 案例：某些topic的Producer的延迟在ack=all的情况下特别高，RemoteTimeMs达到了1s以上，
- 分析：
  - RemoteTimeMs ：等待其他 Broker 完成指定逻辑的时间
  - acks=all，PRODUCE 请求等待ISR完成
  - TotalTimeMs：计算 Request 被处理的完整流程时间。
- 解决：
  - 副本broker ping延迟查过了500ms，网络交换机出现了问题，更换交换机



#### 3 Server处理请求区分优先级

- 案例2：发现删除topic比较慢。怎么分析？
- 分析：删除topic，Controller向Partition leader broker发送 StopRelica请求。
  - 没有被及时处理，操作hang。
  - 数据类和控制类请求不做区分，生产者消息大量积压的broker操作慢。

- 解决：开启数据和控制类请求区分
  - 1个Acceptor，1个Processor，RequestQueue 20

```java
listener.security.protocol.map=CONTROLLER:PLAINTEXT,INTERNAL:PLAINTEXT,EXTERNA
listeners=CONTROLLER://192.1.1.8:9091,INTERNAL://192.1.1.8:9092,EXTERNAL://10.1
control.plane.listener.name=CONTROLLER
```

<img src="0JavaSummary.assets/image-20210702184411993.png" alt="image-20210702184411993" style="zoom:50%;" />



#### 4 分区的 Leader 显示是 -1

- 案例：某些Topic 的Partition Leader 显示是 -1。
  - offlinePartitionCount**该字段统计集群中所有离线或处于不可用状态的主题分区数量**

- 分析：Leader 所在的 Broker 因为负载高宕机， 重启后，Controller 无法分区选举 Leader，因此，“不可用”。
  -  Broker 都在实时监听 ZooKeeper  /controller 节点

    - **监听这个节点是否存在**。不存在，抢着创建 /controller 节点

    - **监听这个节点数据是否发生了变更**。Controller 选举。

- 解决：手动删除了 /controller 。源码中的 **ControllerZNode.path** 上，也就是 ZooKeeper 的 /controller 节点。

```java
// zookeeper 上Controller的临时节点的内容
{"version":1,"brokerid":0,"timestamp":"1585098432431"} // 序号为 0 的 Broker 是集群 Controller。
cZxid = 0x1a
ctime = Wed Mar 25 09:07:12 CST 2020
mZxid = 0x1a
mtime = Wed Mar 25 09:07:12 CST 2020
pZxid = 0x1a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x100002d3a1f0000 // 字段不是 0x0，说明这是一个临时节点。
dataLength = 54
numChildren = 0
```



#### 5 创建Topic，某些Broker不知

- 案例：Kafka 0.10.0.1创建了主题后，有些 Broker 依然无法感知到。
- 分析：元数据变更无法在集群的所有 Broker 上同步，Controller问题
  - UpdateMetadataRequest：更新 Broker 上的元数据缓存。集群上的所有元数据变更，都首先发生在 Controller 端，然后再经由这个请求广播给集群上的所有 Broker。
  - Controller Broker 本身承载着非常重的业务，负载过大，请求积压，造成元数据更新滞后
  - 怀疑是Controller 端的请求积压

- 解决：
  - 源码中新加了一个监控指标，用于实时监控 Controller 的请求队列长度。（**源码是RequestChannel的RequestQueue**），定位了问题。（0.11添加了队列长度和Request在Channel的等待时间`Request Queue Size: kafka.network:type=RequestChannel,name=RequestQueueSize`
  - 迁移Controller到低负载

#### 6 Broker 节点的内存占用高(TODO)

- 案例：Broker 上的副本数过多，Broker 内存占用高。
- 分析：HeapDump ，
- 我们发现根源在于 ReplicaFetcherThread 文件中的 buildFetch 方法。
- 实例化一个 LinkedHashMap。如果分区数很多的话，这个 Map 会被扩容很多次，因此带来了很多不必要的数据拷贝。这样既增加了内存占用，也浪费了 CPU 资源。（初始化16）
- 2.2.0 2019.5 发布，6月2.3.0，fix version: 2.5.0 （April 16, 2020）

```scala
val builder = fetchSessionHandler.newBuilder()

// 改进后
    /** A builder that allows for presizing the PartitionData hashmap, and avoiding making a
     *  secondary copy of the sessionPartitions, in cases where this is not necessarily.
     *  This builder is primarily for use by the Replica Fetcher
     * @param size the initial size of the PartitionData hashmap
     * @param copySessionPartitions boolean denoting whether the builder should make a deep copy of
     *                              session partitions
     */
val builder = fetchSessionHandler.newBuilder(partitionMap.size, false)
```

- 背景：Our current follower replica fetching logic has huge CPU cost with num.partitions to fetch from, and it scales non-linearly as well. There are a bunch of optimizations we can consider to try to reduce its cost and hopefully make it to be linear against the num.partitions.
- PR: Fetch session optimizations (mostly presizing the next hashmap, and avoiding making a copy of sessionPartitions, as a deep copy is not required for the ReplicaFetcher)

> [KAFKA-9039: Optimize ReplicaFetcher fetch path](https://github.com/apache/kafka/pull/7443#)



### Kafka监控

监控什么？机器和进程

> `**Load Average**的值<=**CPU**个数*核数X0.7`，**Load Average**会有3个状态平均值，分别是1分钟、5分钟和15分钟平均**Load**。 如果1分钟平均出现大于**CPU**个数X核数的情况，还不需要担心；如果5分钟的平均也是这样，那就要警惕了；15分钟的平均也是这样，就要分析哪里出现问题。

#### 1 主机监控指标

- 机器负载（Load）
-  CPU 使用率
- 内存使用率，包括空闲内存（Free Memory）和已使用内存（Used Memory）
- 磁盘 I/O 使用率，包括读使用率和写使用率
- 网络 I/O 使用率
- TCP 连接数
- 打开文件数
- inode 使用情况

#### 2 JVM监控

- 搞清楚 Broker 端 JVM 进程的 Minor GC 和 Full GC 的发生频率和时长、活跃对象的总大小和 JVM 上应用线程的大致总数，因为这些数据都是你日后调优 Kafka Broker 的重要依据。
- Full GC的发生频率和时间：看GC log，自己计算频率。
- 存活对象的大小：设置成1.5-2倍成最大堆内存
- 应用线程总数，了解CPU使用情况

#### 3 进程看什么

- 端口能否监听
- 日志有没有异常
- 关键的一些线程是否还存活：
  - kafka-log-cleaner-thread: 日志线程Log Compaction
  - ReplicaFetcherThread : Follower向 Leader 副本拉取消息

#### 4JMX指标

- 网络入口出口：BytesIn\Bytesout， 注意打满
- NetworkProcessorAvgIdlePercent：即网络线程池线程平均的空闲比例。通常来说，你应该确保这个 JMX 值长期大于 30%。如果小于这个值，就表明你的网络线程池非常繁忙，你需要通过增加网络线程数或将负载转移给其他服务器的方式，来给该 Broker 减负。
- RequestHandlerAvgIdlePercent：即 I/O 线程池线程平均的空闲比例。同样地，如果该值长期小于 30%，你需要调整 I/O 线程池的数量，或者减少 Broker 端的负载。
- UnderReplicatedPartitions：即未充分备份的分区数。所谓未充分备份，是指并非所有的 Follower 副本都和 Leader 副本保持同步。一旦出现了这种情况，通常都表明该分区有可能会出现数据丢失。因此，这是一个非常重要的 JMX 指标。
- ISRShrink/ISRExpand：即 ISR 收缩和扩容的频次指标。如果你的环境中出现 ISR 中副本频繁进出的情形，那么这组值一定是很高的。这时，你要诊断下副本频繁进出 ISR 的原因，并采取适当的措施。
- ActiveControllerCount：即当前处于激活状态的控制器的数量。正常情况下，Controller 所在 Broker 上的这个 JMX 指标值应该是 1，其他 Broker 上的这个值是 0。如果你发现存在多台 Broker 上该值都是 1 的情况，一定要赶快处理，处理方式主要是查看网络连通性。这种情况通常表明集群出现了脑裂。脑裂问题是非常严重的分布式故障，Kafka 目前依托 ZooKeeper 来防止脑裂。但一旦出现脑裂，Kafka 是无法保证正常工作的。

#### 5 监控Kafka客户端

- 生产者需要监控什么？有没有在正常工作，kafka-producer-network-thread ; 
- Producer : request-latency，即消息生产请求的延时。
- Consumer : records-lag 和 records-lead 
- Consumer Group: join rate 和 sync rate， Rebalance 的频繁程度。

> 监控框架：Kafka Manager，以didi的最为流行了，监控管理、 Kafka Eagle 

### 实际操作

- 监控 Kafka
  - Kafka Manager、Kafka Monitor、JMX 监控、JMXTool
  
- Broker 的 Heap Size 如何设置
  - 稳定后，手动触发(jmap)Full GC，存活对象的 1.5~2 倍。 6GB。
  
- 估算 Kafka 集群的机器数量？
  
  - 带宽
  - 磁盘
  - 48G内存，CPU 24核数 
  
- Kafka某个Topic的分区数量如何确定？需要考虑你的目标是什么？

  - 比如Producer的TPS是10万条/秒，记为T1, 然后在真实环境中，创建仅有1个分区的topic，往里面写，看看TPS=T2，这个就是每个分区能够写入的最大条数。然后分区数=T1/T2。（简单有效。）

- JVM参数：

  ```bash
  export KAFKA_HEAP_OPTS=--Xms6g  --Xmx6g
  export  KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true
  bin/kafka-server-start.sh config/server.properties
  ```


### Pulsar

- 与Kafka的不同：存储计算分离

  - Broker Stateless，无状态
    - Broker与分区对应是动态调整的
  - ZK存储元数据，和Kafka一样
  - Bookeeper，分布式存储集群，存储消息
    - Ledger，是Write Ahead Log，类似于Segment，但是是一次性写入（解决并发写入控制，不需要分布式锁，不需要损失性能）

- <img src="0JavaSummary.assets/image-20210723104938142.png" alt="image-20210723104938142" style="zoom: 80%;" />

- 客户端如何读写消息

  - 连接Service Discovery，获取分区与Broker的元数据信息
  - 连接对应的Broker

- 存储分离优点

  - 复杂度降低
  - 计算节点无状态，扩展、故障转移快
  - 计算节点：只关注业务逻辑，调度灵活
  - 存储节点：只关注存储

- 存储分离缺点

  - BookKeeper 依然要解决数据一致性、节点故障转移、选举、数据复制等等这些问题
  - 单集群变多集群，运维复杂
  - 性能损失，比如消费一条消息，Broker需要从Bookeeper读取，多了网络IO和内存拷贝

  

## Linux 

<img src="0JavaSummary.assets/image-20210720222748204.png" alt="image-20210720222748204" style="zoom:33%;" />

- 内核：特殊程序，控制所有硬件资源，如CPU、内存
- 用户态：应用程序运行的空间
- 系统调用：内核态为用户态提供访问资源的接口。
  - 查看1个文件过程：
    -  open() 打开文件，用户态
    -  read() 读取内容，内核态（上下文切换1次）
    - write() 写到标准输出，用户态（上下文切换1次）
    -  close() 关闭文件，用户态

- 用户态到内核态切换方式：
  - 系统调用，系统调用是软中断
  - 异常：当前进程运行在用户态发生异常，如：缺页异常。
  - 外设中断：当外设完成用户请求时，向CPU发送中断信号。

- CPU 上下文切换: 把上一个任务的 CPU 上下文（ CPU 寄存器和程序计数器）保存到系统内核，然后加载新任务的上下文，最后根据程序计数器运行新任务。
- Mutex互斥锁的实现：CPU指令，swap和exchange

参考：[一文让你明白CPU上下文切换](https://zhuanlan.zhihu.com/p/52845869)

### Top

- **us：用户态使用的cpu时间比**
- **sy：系统态使用的cpu时间比**
- ni：用做nice加权的进程分配的用户态cpu时间比
- id：空闲的cpu时间比
- **wa：cpu等待磁盘写入完成时间（高，磁盘忙，IO问题，iostat排查）**
- hi：硬中断消耗时间
- **si：软中断消耗时间**（高，网卡中断到某个CPU，巨大网络流量，造成网卡的数据包收发出现延迟。解决：网卡binding到多CPU）
- st：虚拟机偷取时间

### 进程线程

1. 是什么，区别？
   - 进程是资源分配的最小单位
   - 线程是CPU调度的最小单位。
   - 从系统开销来看，哪个大？进程
2. 进程的状态和操作系统是如何调度的，有哪些调度算法？
   - 有5种状态，能否解释一下僵尸进程是如何产生的，孤儿进程是如何产生的。
   - 先来先服务，短作业有限，时间片，优先级
3. 进程间如何通信？线程呢？
   - 进程有很多：共享内存、消息队列、pipe、Socket、共享存储、信号量
   - 线程：共享内存，尤其是在Java里面

### 内存管理

管什么？物理内存和虚拟内存。

虚拟内存是什么，用来做什么？磁盘的一块区域，有最大的大小限制。目的是扩大整个逻辑内存。

操作系统是怎么管理程序的？给程序分配地址空间，每一个就是一页，把页映射到物理内存和虚拟内存。程序运行的时候，会引用数据，这个数据如果在物理内存页，则直接运行；如果在虚拟内存页，就把虚拟内存的内容置换到物理内存，然后运行（缺页）。

> 在程序运行过程中，如果要访问的页面不在内存中，就发生缺页中断从而将该页调入内存中。此时如果内存已无空闲空间，系统必须从内存中调出一个页面到磁盘对换区中来腾出空间。

- 页面置换算法干什么用？尽可能减少虚拟内存和物理内存的置换。LRU出名。

### TCP 建立

为什么需要三次握手？通过三次握手才能阻止重复历史连接的初始化，导致混乱。

将重点放到为什么需要 TCP 建立连接需要**『三次握手』**，而*不仅仅*是为什么需要**『三次』**握手。

- 我们对于『什么是连接』真的清楚？只有知道**连接的定义**，我们才能去尝试回答为什么 TCP 建立连接需要三次握手。(RFC793翻译过来：用于保证可靠性和流控制机制的信息，包括 Socket、序列号以及窗口大小叫做连接。)
- 到这里，我们将原有的问题转换成了『为什么需要通过三次握手才可以初始化 Sockets、窗口大小和初始序列号？』
  * Socket是建立连接用的
  * 初始序列号 可以用来传输数据的去重、重传等
  * 窗口大小：

使用三次握手和 `RST` 控制消息将是否建立连接的最终控制权交给了发送方，因为只有发送方有足够的上下文来判断当前连接是否是错误的或者过期的，这也是 TCP 使用三次握手建立连接的最主要原因。

<img src="0JavaSummary.assets/image-20210720194136145.png" alt="image-20210720194136145" style="zoom:50%;" />

如下图所示，通信双方的两个 `TCP A/B` 分别向对方发送 `SYN` 和 `ACK` 控制消息，等待通信双方都获取到了自己期望的初始化序列号之后就可以开始通信了，由于 TCP 消息头的设计，我们可以将中间的两次通信合成一个，`TCP B` 可以向 `TCP A` 同时发送 `ACK` 和 `SYN` 控制消息，这也就帮助我们将四次通信减少至三次。

<img src="0JavaSummary.assets/image-20210720194216009.png" alt="image-20210720194216009" style="zoom:50%;" />

> 为什么DNS查询使用UDP协议？盲猜：历史原因，数据包小，tcp连接消耗太大。不同的实现场景可以用TCP来实现DNS查询。

### 为什么TCP协议有Time_Wait状态

> TCP断开连接，一定是从某一端发起的，暂且认为是客户端发起的，因为服务端都是被动的。

使用 TCP 协议通信的双方会在关闭连接时触发 `TIME_WAIT` 状态，关闭连接的操作其实是告诉通信的另一方**自己没有需要发送的数据**，但是它仍然**保持了接收对方数据的能力**，一个常见的关闭连接过程如下[1](https://draveness.me/whys-the-design-tcp-time-wait/#fn:1)：

1. 当客户端没有待发送的数据时，它会向服务端发送 `FIN` 消息，发送消息后会进入 `FIN_WAIT_1` 状态；
2. 服务端接收到客户端的 `FIN` 消息后，会进入 `CLOSE_WAIT` 状态并向客户端发送 `ACK` 消息，客户端接收到 `ACK` 消息时会进入 `FIN_WAIT_2` 状态；
3. 当服务端没有待发送的数据时，服务端会向客户端发送 `FIN` 消息；
4. 客户端接收到 `FIN` 消息后，会进入 `TIME_WAIT` 状态并向服务端发送 `ACK` 消息，服务端收到后会进入 `CLOSED` 状态；
5. 客户端等待**两个最大数据段生命周期**（Maximum segment lifetime，MSL）的时间后也会进入 `CLOSED` 状态；

<img src="0JavaSummary.assets/image-20210720194411412.png" alt="image-20210720194411412" style="zoom:50%;" />

**图 2 - TCP 关闭连接的过程** 

> 为什么要四次挥手的原因是 TIME WAIT状态为什么要存在。

从上述过程中，我们会发现 `TIME_WAIT` 仅在主动断开连接的一方出现，被动断开连接的一方会直接进入 `CLOSED` 状态，进入 `TIME_WAIT` 的客户端需要等待 2 MSL 才可以真正关闭连接。TCP 协议需要 `TIME_WAIT` 状态的原因和客户端需要等待两个 MSL 不能直接进入 `CLOSED` 状态的原因是一样的[3](https://draveness.me/whys-the-design-tcp-time-wait/#fn:3)：

- 因为数据段的网络传输时间不确定，所以可能会收到上一次 TCP 连接中未被收到的数据段；
- 因为客户端发出的 `ACK` 可能还没有被服务端接收，服务端可能还处于 `LAST_ACK` 状态，所以它会回复 `RST` 消息终止新连接的建立；

<img src="0JavaSummary.assets/image-20210720194543855.png" alt="image-20210720194543855" style="zoom:50%;" />

<img src="0JavaSummary.assets/image-20210720194607367.png" alt="image-20210720194607367" style="zoom:50%;" />

参考：[为什么 TCP 协议有 TIME_WAIT 状态](https://draveness.me/whys-the-design-tcp-time-wait/)

### TCP 拥塞控制

问几个问题：

1. TCP是如何保证可靠传输的？ 它可以超时重传，可以利用seq来判断。
2. 什么是滑动窗口？Client发送方和Server接收方都有一个窗口，来记录缓存。
3. 什么是TCP的流量控制
4. TCP的拥塞控制是为了解决什么问题？用的是什么算法，慢开始--拥塞避免--快重传-快恢复。

![image-20210720194808310](0JavaSummary.assets/image-20210720194808310.png)

> TCP流量控制是控制谁，为了谁好，如何实现的？

它是控制发送方的发送速度，为了接收方来得及接受。

它是通过接收方的窗口字段来告诉发送方的，比如设置为0，表示接收方不接受数据，那么发送方就不会发送数据。

#### 解决什么？

解决网络出现拥塞的时候，发送方和接收方应该怎么办？

如果网络出现拥塞，分组将会丢失，此时发送方会继续重传，从而导致网络拥塞程度更高。因此当出现拥塞时，应当控制发送方的速率。这一点和流量控制很像，但是出发点不同。

> 注意与流量控制进行区别：流量控制是为了让接收方能来得及接收，而拥塞控制是为了降低整个网络的拥塞程度。

<img src="0JavaSummary.assets/image-20210720194839718.png" alt="image-20210720194839718" style="zoom:50%;" />



> TCP 主要通过四个算法来进行拥塞控制：慢开始、拥塞避免、快重传、快恢复。

发送方需要维护一个叫做**拥塞窗口（cwnd）的状态变量**，注意拥塞窗口与发送方窗口的区别：拥塞窗口只是一个状态变量，实际决定发送方能发送多少数据的是发送方窗口。

为了便于讨论，做如下假设：

- 接收方有足够大的接收缓存，因此不会发生流量控制；
- 虽然 TCP 的窗口基于字节，但是这里设窗口的大小单位为报文段。

![image-20210720194907891](0JavaSummary.assets/image-20210720194907891.png)

####  1慢开始与拥塞避免（发送方角度）

- 发送的最初执行慢开始，令 cwnd = 1，发送方只能发送 1 个报文段；当收到确认后，将 cwnd 加倍，因此之后发送方能够发送的报文段数量为：2、4、8 ...

- 慢开始每个轮次都将 cwnd 加倍，这样会让 cwnd 增长速度非常快，从而使得发送方发送的速度增长速度过快，网络拥塞的可能性也就更高。设置一个**慢开始门限 ssthresh**，当 cwnd >= ssthresh 时，进入拥塞避免，每个轮次只将 cwnd 加 1。

- 如果出现了超时，则怎么更新**慢开始的门限ssthresh** ？减半。ssthresh = cwnd / 2，然后重新执行慢开始cwnd=1。

#### 2. 快重传与快恢复

- 在接收方，要求每次接收到报文段都应该对最后一个已收到的有序报文段进行确认。例如已经接收到 M1 和 M2，此时收到 M4，应当发送对 M2 的确认。

- 在发送方，如果收到3个重复确认，那么可以知道下一个报文段丢失，此时执行**快重传**，立即重传下一个报文段。例如收到三个 M2，则 M3 丢失，立即重传 M3。（为什么是3个，因为认为报名最多存活2个Maxium Segment Lifetime）

- 在这种情况下，只是丢失个别报文段，而不是网络拥塞。因此执行**快恢复**，(减半慢开始的门限值，并且直接进入拥塞避免)令 ssthresh = cwnd / 2 ，cwnd = ssthresh，注意到此时直接进入拥塞避免。

慢开始和快恢复的快慢指的是 cwnd 的设定值，而不是 cwnd 的增长速率。慢开始 cwnd 设定为 1，而快恢复 cwnd 设定为 ssthresh。

### 零拷贝

> "**Zero-copy**" describes computer operations in which the CPU does not perform the task of copying data from one memory area to another. This is frequently used to save CPU cycles and memory bandwidth when transmitting a file over a network.

是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域，这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。

实现的两种方式分别是：

- **mmap+write**
- **Sendfile**

Java进程发起Read/Write请求加载数据的大致流程：底层调用Linux `read() write()`实现

<img src="0JavaSummary.assets/3b0c6e94c9cc493c8d1cad2e07b9ddb0tplv-k3u1fbpfcp-zoom-1.image" alt="优享资讯| 什么是mmap？ 经典题目" style="zoom:67%;" />

这个过程有什么问题? 一次简单的IO过程产生了4次上下文切换，在高并发场景下会对性能产生较大的影响。

- 用户进程通过`read()`方法向操作系统发起调用，此时上下文从用户态转向内核态
- DMA控制器把数据从硬盘中拷贝到读缓冲区
- CPU把读缓冲区数据拷贝到应用缓冲区，上下文从内核态转为用户态，`read()`返回
- 用户进程通过`write()`方法发起调用，上下文从用户态转为内核态
- CPU将应用缓冲区中数据拷贝到socket缓冲区
- DMA控制器把数据从socket缓冲区拷贝到网卡，上下文从内核态切换回用户态，`write()`返回

> DMA（Direct Memory Access）直接内存访问技术，本质上来说他就是一块主板上独立的芯片，通过它来进行内存和IO设备的数据传输，从而减少CPU的等待时间。



**利用虚拟内存，让内核空间和用户空间的虚拟地址，映射到同一个物理内存。这样DMA填充这块缓冲区的时候，两个空间都可见。**

![clip_image004](0JavaSummary.assets/1550970-20201212172033717-1748604803-1624862837723.png)

#### mmap + write

利用虚拟内存，让内核空间和用户空间的虚拟地址，映射到同一个物理内存。

使用`mmap`替换了read+write中的read操作，减少了一次CPU的拷贝。

- mmap 是一种内存映射文件的方法

- 换一种说法，实现方式是将读缓冲区的地址和用户缓冲区的地址进行映射，内核缓冲区和应用缓冲区共享。

<img src="0JavaSummary.assets/b306256e43ba468abd5137775769cac1tplv-k3u1fbpfcp-zoom-1.image" alt="img" style="zoom:80%;" />



整个过程发生了**4次用户态和内核态的上下文切换**和**3次拷贝**，具体流程如下：

1. 用户进程通过`mmap()`方法向操作系统发起调用，上下文从用户态转向内核态
2. DMA控制器把数据从硬盘中拷贝到读缓冲区
3. **上下文从内核态转为用户态，mmap调用返回**
4. 用户进程通过`write()`方法发起调用，上下文从用户态转为内核态
5. **CPU将读缓冲区中数据拷贝到socket缓冲区**
6. DMA控制器把数据从socket缓冲区拷贝到网卡，上下文从内核态切换回用户态，`write()`返回

>  适用场景：`mmap`的方式下，用户进程中的内存是虚拟的，只是映射到内核的读缓冲区，所以可以节省一半的内存空间，比较适合大文件的传输。



#### sendfile

替代了`read+write`，减少一次CPU拷贝，和2次上下文切换

它是linux2.1引入的系统调用函数，目的是简化网络在两个通道之间的数据传输过程。`sendfile`替代了`read+write`，减少了数据复制，节省了一次系统调用，也就是2次上下文切换。

<img src="0JavaSummary.assets/d6d68cc34030404a85d30d39c60ab3e4tplv-k3u1fbpfcp-zoom-1.image" alt="img" style="zoom:80%;" />



整个过程发生了**2次用户态和内核态的上下文切换**和**3次拷贝**，具体流程如下：

1. 用户进程通过`sendfile()`方法向操作系统发起调用，上下文从用户态转向内核态
2. DMA控制器把数据从硬盘中拷贝到读缓冲区
3. CPU将读缓冲区中数据拷贝到socket缓冲区
4. DMA控制器把数据从socket缓冲区拷贝到网卡，上下文从内核态切换回用户态，`sendfile`调用返回

>  `sendfile`方法IO数据对用户空间完全不可见，所以只能适用于完全不需要用户空间处理的情况，比如静态文件服务器。

>  更高级： 数据传送只发生在内核空间，所以减少了一次上下文切换；但是还是存在一次 Copy，能不能把这一次 Copy 也省略掉？
>
>  Linux2.4内核优化，将 Kernel buffer 中对应的数据描述信息（内存地址，偏移量）记录到相应的 Socket 缓冲区当中，节约CPU copy。（这种叫做DMA gather）

它将读缓冲区中的数据描述信息--内存地址和偏移量记录到socket缓冲区，由 DMA 根据这些将数据从读缓冲区拷贝到网卡，相比之前版本减少了一次CPU拷贝的过程

<img src="0JavaSummary.assets/a97c4913546e4ee38f9a44cbec0b7a97tplv-k3u1fbpfcp-zoom-1.image" alt="img" style="zoom:67%;" />

整个过程发生了**2次用户态和内核态的上下文切换**和**2次拷贝**，其中更重要的是完全没有CPU拷贝，具体流程如下：

1. 用户进程通过`sendfile()`方法向操作系统发起调用，上下文从用户态转向内核态
2. DMA控制器利用scatter把数据从硬盘中拷贝到读缓冲区离散存储
3. CPU把读缓冲区中的文件描述符和数据长度发送到socket缓冲区
4. DMA控制器根据文件描述符和数据长度，使用scatter/gather把数据从内核缓冲区拷贝到网卡
5. `sendfile()`调用返回，上下文从内核态切换回用户态

#### 中间件的应用

- RocketMQ：生产者和消费者都是`mmap+write`
- Kafka: 生产者是`mmap+write`, 消费者或者Follow同步消息是`sendfile`
- Netty：



#### Java 零拷贝实现

- MappedByteBuffer 实现mmap，底层是生成了**DirectByteBuffer**，堆外内存。
- FileChannel的transferTo实现 **Channel-to-Channel **，实现了sendFile

```java

// FileChannel transferTo：开始传输的位置，传输的字节数，以及目标通道
public abstract long transferTo(long position, long count, WritableByteChannel target) throws IOException;
// 用法：channel.transferTo(0, channel.size(), target);

```



#### 总结

- 由于CPU和IO速度的差异问题，产生了DMA技术，通过DMA搬运来减少CPU的等待时间。

- 传统的IO`read+write`方式会产生2次DMA拷贝+2次CPU拷贝，同时有4次上下文切换。

- 而通过`mmap+write`方式则产生2次DMA拷贝+1次CPU拷贝，4次上下文切换，通过内存映射减少了一次CPU拷贝，可以减少内存使用，适合大文件的传输。

- `sendfile`方式是新增的一个系统调用函数，产生2次DMA拷贝+1次CPU拷贝，但是只有2次上下文切换。因为只有一次调用，减少了上下文的切换，但是用户空间对IO数据不可见，适用于静态文件服务器。

- `sendfile+DMA gather`方式产生2次DMA拷贝，没有CPU拷贝，而且也只有2次上下文切换。虽然极大地提升了性能，但是需要依赖新的硬件设备支持。

参考：[简单易懂](https://blog.csdn.net/zhengchao1991/article/details/104524468) [大神的Linux I/O 原理和 Zero-copy 技术全面揭秘](https://mp.weixin.qq.com/s/TEUrcD4c_8Aw7bzTXr83kw)

## Storm

- storm总体结构

  - stream 是被处理的数据，spout 是数据源，bolt 封装了数据处理逻辑
  - worker 是工作进程，一个工作进程可包含多个 Executor 线程，Executor 是运行 spout 或bolt的处理逻辑线程，task是storm中的最小处理单元，一个executor中可包含多个task，消息分发都是从一个task到另一个task进行的
  - stream grouping 定义了消息分发策略，定义了 bolt 节点以什么方式接收数据（shuffle grouping、fields grouping、all grouping、global grouping、none grouping、diirect grouping）
  - topology 是一整个任务拓扑，是由于消息分组方式连接起来的 spout 和 bolt 网络

  <img src="0JavaSummary.assets/120049.png" alt="img" style="zoom: 33%;" />

  - Nimbus 和 Supervisor 无状态，可快速恢复，源数据存储在ZK中，worker 由 supervisor 启动，topo 由 nimbus 启动
  - 特殊的系统 bolt：ACKer 和 Metrics

- 通信机制

  - 进程内用Disruptor
    - 每个 worker 有一个独立的接收线程、发送线程，接收线程负责从接收缓冲区将数据转发至各个 Executor的 incoming-queue，发送线程则负责从 Disuptor transfer queu e读取数据发送至网络
    - 每个 executor 有一个单独的线程处理 spout/bolt 逻辑，处理完的 tuple 被放入 out-queue 中，out-queue 中的 tuple 积累到一定的阈值后，send thread 从中取出放入 worker 的 shared transfer queue（disruptor）中等待发送

  ![img](0JavaSummary.assets/120054.png)

  - 进程间用 ZMQ/Netty

  <img src="0JavaSummary.assets/120052.png" alt="img" style="zoom:67%;" />

- ACK

  - ACKer 是一个 bolt，其维护了一个 Map 存储了 root-id 到 tuple 树相关信息的映射 `{root-id {:spout-task task-id :val ack-val :failed bool-val …}}`
  - 当 spout 创建一条新 tuple 时，会给 ACKer 发送消息，bolt 接收到该 tuple 后调用 ack 给 ACker 发送搞一条消息，同时 bolt 如果也产生了新的 tuple 往后续 bolt 发送，那么也将执行类似前面的逻辑，因此最后所有的 tuple-id 都会被异或两次，从而使得 ack-val 最中归零

  <img src="0JavaSummary.assets/image-20210727205636125.png" alt="image-20210727205636125" style="zoom: 33%;" />

- 反压机制

  - 如果 executor 发现 recv queue 负载超过高水位值（high watermark）则通知反压线程（backpressure thread）
  - 反压线程将反压信息写到 Zookeeper
  - Zookeeper 上的 watch 会通知该拓扑（topo）的所有 Worker，该拓扑出现反压
  - Spout 减缓发送 tuple 的速率

## 项目

###  内存泄露

- 背景
  - Test service上线10多天，服务接口无法reponse或者response时间太长，堆大小设置是4G. 

- 分析
  - top free df
    - CPU 500%，正常情况下，不到几十
    - 死循环，or 大量Full GC.
  - jstack 分析线程
  - jstat -gc pid 
    - 1s一次Full GC，推断内存泄露
  - `jmap -dump:format=b,file=heap.log pid`, MAT工具分析，选择内存泄露
    - 技巧：堆文件太大，gzip压缩，推荐-6.
    - JSONObject对象占用 96%
    - 项目里全局搜索对象名， Bean 对象，然后定位到 Map 。
  - 原因分析：
    - JSONObject是每次探测接口响应的结果
    - 每次探测完都塞到 ArrayList 里去分析
    - ArrayList 又被存储到Map
    - 由于 Bean 对象不会被回收，这个属性没有清除逻辑，所以时间越长，这个 Map 越来越大，直至将内存占满。
  - 解决
    - 业务改进，Map分析完，添加清除逻辑

### 内存浪费

- 背景
  - 流式计算集群60台，内存2.6T，占用了2.5T。业务发展，需要扩容
  - 流式应用的内存资源浪费严重
- 立项，定义目标
  - 能否不扩容，提高内存使用率
  - 目前的内存使用率：低的达到30，高的有60%
  - 目标使用率80%，核心业务：60%
- 实施
  - 分析内存资源浪费原因
    - 用户原因：用户申请的内存资源不合理（主要原因）
    - OLS本身不在增加内存。
    - Yarn程度的Memory浪费(次要原因)
  - 解决：
    - 用户：按需分配
    - OLS: 不在加倍1.25
    - Yarn：1G改为128M为最小的分配单位，细粒度分配资源。
      - 内存资源：**进程监控方案**，内存过量的标准如下：
      - 如果一个Container对应的所有进程（年龄大于0）总内存超过最大值的2倍

      - 或者所有年龄大于1的进程总内存量超过最大值	
- 结果
  - 提高内存使用率到80%，平均使用率65%
  - 节约了1T的内存空间，不需要扩容机器

### 工作挑战

* S: Loris系统，第一版本后，核心成员相继离职，我们接受这个项目，一来就是性能问题
  * 用户登陆慢，LDAP Server
  * 查询慢，并且不准确
  * 查看历史测试结果，慢30s
  * 后来，Test service内存泄露
* T: 分页面解决核心问题
  * 登陆页面
  * Inventory页面
  * Workflow test case 页面
  * Test Service页面
* A: Manager是上海，remote，文档和代码，熟悉文档、代码、发布流程、业务。细化问题，突出优先级，分模块
* R:Invertery、Reservation服务提升，用户满意度提高。半个月熟悉，不到一个月上线调优，服务质量提升。

### 增量构建

- 背景
  - 所有代码集中在一个repository，20+服务
  - 开发改动代码，自己测试一下，需要所有服务重新部署，20分钟
- 立项、定义目标
  - 缩短部署服务时间，能否只部署改动的服务
- 实施
  - 分析pom文件，解析项目间依赖关系，生成依赖关系图
  - git revision 可以知道某个目录是否发生更改，把revision信息记录到docker image的tag里面。
  - 这样，当一个服务修改后，只需要重新部署依赖它的相关服务。
  - 实现了项目的增量构建
- 结果
  - 平均缩短时间为2分钟，节约90%的构建部署时间，提升开发测试效率。

### 幂等性设计

**对于一个接口，如何做幂等性设计？**

- 核心思想是利用唯一的ID来标识某一种操作，比如下单。
  - mysql自增ID可以实现
  - 分布式唯一ID可以实现
  - 前端可以利用token过滤一些不必要的重试操作

**如何在分布式系统生成唯一的ID**

- UUID，一组32位数的16进制数字，相对比较长，无序的。
- snowflake：时间有序

**snowflake的结构**

长度是1个long型的数字，64位。**snowflake的结构如下(每部分用-分开):**

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
```

- 第1位为未使用，
- 41bit 保存时间戳，精确到毫秒。最大可使用的年限是69年。
- 10bit 的机器位，能部属在1024台机器节点来生成ID。
- 12bit 的序列号，一毫秒最大生成唯一ID的数量为4096个。

> 一共加起来刚好64位，为一个Long型。(转换成字符串长度为18).
>
> snowflake生成的ID整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞（由datacenter和workerId作区分），并且效率较高。据说：snowflake每秒能够产生26万个ID。

Code : [snow flake](https://github.com/HQebupt/app/blob/master/src/main/java/design/SnowFlake.java)

- 缺点
  - 依赖机器时钟，如果时钟回拨，导致生成ID重复，打破递增属性
    - 如果发现有时钟回拨，时间很短比如5毫秒,就等待，然后再生成。或者就直接报错，交给业务层去处理。
    - 放入到Redis缓存

## 杂货

### 监控平台Prometheus

| Zabbix                                                       | Prometheus                                                   |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 后端用 C 开发，界面用 PHP 开发，定制化难度很高。             | 后端用 golang 开发，前端是 Grafana，JSON 编辑即可解决。定制化难度较低。 |
| 集群规模上限为 10000 个节点。                                | 支持更大的集群规模，速度也更快。                             |
| 更适合监控物理机环境。                                       | 更适合云环境的监控，对 OpenStack，Kubernetes 有更好的集成。  |
| 监控数据存储在关系型数据库内，如 MySQL，很难从现有数据中扩展维度。 | 监控数据存储在基于时间序列的数据库内，便于对已有数据进行新的聚合。 |
| 安装简单，zabbix-server 一个软件包中包括了所有的服务端功能。 | 安装相对复杂，监控、告警和界面都分属于不同的组件。           |
| 图形化界面比较成熟，界面上基本上能完成全部的配置操作。       | 界面相对较弱，很多配置需要修改配置文件。                     |
| 发展时间更长，对于很多监控场景，都有现成的解决方案。         | 2015 年后开始快速发展，但发展时间较短，成熟度不及 Zabbix。   |
| Push 模型（客户端发送数据给服务端）                          | Pull 模型在云原生环境中有比较大的优势，                      |

- 客户端使用push的方式上报监控数据到pushgateway，prometheus会定期从pushgateway拉取数据。使用它的原因主要是:Prometheus 采用 pull 模式，可能由于不在一个子网或者防火墙原因，导致Prometheus 无法直接拉取各个 target数据
- Counter 
  - Counter 用于累计值，例如 记录 请求次数、任务完成数、错误发生次数。
  - 一直增加，不会减少。
  - 重启进程后，会被重置。
- Gauge
  - Gauge 常规数值，例如 温度变化、内存使用变化。
  - 可变大，可变小。
  - 重启进程后，会被重置
- Histogram 柱状图
  - 常用于跟踪事件发生的规模，例如：请求耗时、响应大小。它特别之处是可以对记录的内容进行分组，提供 count 和 sum 全部值的功能。例如：{小于10=5次，小于20=1次，小于30=2次}，count=8次，sum=8次的求和值
- Summary
  - Summary和Histogram十分相似，常用于跟踪事件发生的规模，例如：请求耗时、响应大小。同样提供 count 和 sum 全部值的功能。
    例如：count=7次，sum=7次的值求值
    它提供一个quantiles的功能，可以按%比划分跟踪的结果。例如：quantile取值0.95，表示取采样值里面的95%数据。

- **summary和histogram的选择** 
  - Summary 结构有频繁的全局锁操作，对高并发程序性能存在一定影响。histogram仅仅是给每个桶做一个原子变量的计数就可以了，而summary要每次执行算法计算出最新的X分位value是多少，算法需要并发保护。会占用客户端的cpu和内存。
  - 不能对Summary产生的quantile值进行aggregation运算（例如sum, avg等）。例如有两个实例同时运行，都对外提供服务，分别统计各自的响应时间。最后分别计算出的0.5-quantile的值为60和80，这时如果简单的求平均(60+80)/2，认为是总体的0.5-quantile值，那么就错了。
  - summary的百分位是提前在客户端里指定的，在服务端观测指标数据时不能获取未指定的分为数。而histogram则可以通过promql随便指定，虽然计算的不如summary准确，但带来了灵活性。
  - histogram不能得到精确的分为数，设置的bucket不合理的话，误差会非常大。会消耗服务端的计算资源。

所以对比得到的总结是：

1. 如果需要聚合（aggregate），选择histograms。
2. 如果比较清楚要观测的指标的范围和分布情况，选择histograms。如果需要精确的分为数选择summary。

### K8S

![image-20210714172117556](0JavaSummary.assets/image-20210714172117556.png)

- Master：集群的控制和管理
  - ApiServer：提供集群管理的授权、访问控制、发现、认证等功能
  - Scheduler：调度器，收集每个Worker资源的详细信息及运行情况，方便调度决策。
  - Controller Manager：监视集群状态，Pod副本管理。
- Worker：工作节点
  - Kuberlet：管理Pod的生命周期
  - Container Runtime：运行时环境，比如Docker
  - kube-proxy：代理，转发Service的请求到Pod
- Etcd：存储集群的元数据信息
- 概念
  - Pod：最小的调度单位
  - ReplicaSet：管Pod
  - Deployment：管理Pod、ReplicaSet
  - Service：
