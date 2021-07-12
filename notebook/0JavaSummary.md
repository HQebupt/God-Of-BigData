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

### 1常见的Java问题

- ArrayList, LinkedList, HashSet, HashMap, CopyOnWriteList, ConcurrentHashMap, ConcurrentLinkedQueue, volatile, Atomic, CAS, double check locking, happen-before, Object Header, false-sharing, ThreadExecutor(Cached, Fixed, Scheduled), syncronize, Lock, CountdownLatch, Barrier, Exchanger, JVM survivor, GC Algorithem, JVM tuning, syncronize tuning in JDK1.6, strong/weak/phantom reference, String pool

#### 1.1 HashMap

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
#### 2.2 ConcurrentHashMap

> 漏掉了CopyOnWriteArrayList，后期有时间，补上。TODO

@jdk7 使用分段锁设计；iterator 不会抛 concurrentModification 异常，但是只能被一个线程操作；size 方法开销很大 

![img](0JavaSummary.assets/120290-1622708615286)

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



@jdk8 改为 CAS 设计，更细力度地控制锁，①对于一个空的Node，CAS无锁添加；对于非空的Node，对Node加锁（synchronized，锁可以被优化）；增加 addCount 方法专门记录 size ；并发修改一个下标的 Node 时才加 synchronized ，并且只锁定当前的 Node

- 取值操作不阻塞，因此与更新操作会有所重叠；取值操作反映最近正好已完成的更新，即这种反映遵循一种 happen-before 的关系；批量操作时，并发取值操作操作反映部分的插入、移除的结果，而非批量操作的整体完成后的结果；同理，迭代器操作反映的是迭代器创建那一刻的结果集
- isEmpty、size、containsValue 只在 map 不并发更新的时候准确，适合用于监控或估算，而不适于程序控制；不支持 Null Key/Value
- putVal
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

- get操作：比较简单，就是用key的hash值去命中Node。没有加锁，使用volatile来保证可见性。这就是所谓的弱一致性。

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

##### size

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

#### 四种引用

- 强引用:无论什么时候都不会自动回收（场景：正常使用）
- 软引用:空间不足才会回收对象（场景：适合缓存）
- 弱引用:不是立刻回收而是GC发现后会在下次GC时才会回收对象。（场景：如果对象偶尔使用可WeakReference.）
- 虚引用:必须和引用队列(ReferenceQueue)联合使用,本次GC发现立即回收对象（场景：GC里面使用？）

StrongReference、WeakReference、SoftReference、PhantomReference

- WeakReference 引用的对象，当没有强引用指向它后，将在 GC 时被回收；如果其作为 Map.Entry 中的key，则整个 Entry 会被移除。

```java
    Obejct reference = new Object();
    WeakReference<Obejct> weakRef = new WeakReference<>(reference);

    reference = null;
    System.gc(); // 被回收
    AssertNull(weakRef.get())
```

- SoftReference 与 WeakReference 特性类似，区别在于 SoftReference 被回收的时机是在 JVM 内存不足之时；因此适合用于做缓存

```java
		String str = new String("abc");
		SoftReference<String> softReference = new SoftReference<String>(str);
		// 浏览器的回退可以用缓存设计：
    if(softReference.get() != null) {
        page = softReference.get(); // 内存充足，还没有被回收器回收，直接获取缓存
    } else {
        page = browser.getPage();// 内存不足，软引用的对象已经回收
        softReference = new SoftReference(page);// 重新构建软引用
    }
```

- PhantomReference，调用 get 永远返回 null，用于跟踪引用何时被 enqueue 至 ReferenceQueue 中.**虚引用必须和引用队列(ReferenceQueue)联合使用**。

  > 对于GC来看，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，会在回收之前，把虚引用加入到与之关联的引用队列中。
  >
  > 对于用户来看，如果程序发现某个虚引用已经被加入到ReferenceQueue，那么回收之前采取一些行动。

```java
    String str = new String("abc");
    ReferenceQueue queue = new ReferenceQueue();
    // 创建虚引用，要求必须与一个引用队列关联
    PhantomReference pr = new PhantomReference(str, queue);
```

## 3 JVM

<img src="0JavaSummary.assets/image-20210712082721625.png" alt="image-20210712082721625" style="zoom: 33%;" />

### 结构、回收算法

- 基本结构：程序计数器、JVM 栈、native 栈、堆、运行时常量池
- 分区：垃圾收集器把堆分为新生代、老年代、永久代，1.8 版本后引入了 MetaSpace 替换永久代，本地内存。
- 直接内存: NIO，`DirectByteBuffer分配
- 基本回收算法：标记清除，标记整理，复制

- 对象分配策略
  - 优先分配在新生代
  - 大对象直接老年代(-XX:PretenureSizeThreshold)
  - 长时间存活对象进入老年代(- XX:MaxTenuringThreshold)
  - 若 Survivor 中相同年龄的对象大小和 > Survivor 空间的一半，则年龄大于等于该值的对象直接进入老年代
  - 空间分配担保策略
    - YGC 前 JVM 检查老年代的连续空间是否大于新生代所有对象总和
    - 若上述为否且 HandlePromotionFailure 为 true ，则检查老年代连续空间是否大于每次晋升对象的平均大小
    - 若上述为否，或 HandlePromotionFailure 为 false ，则触发 FullGC

- GC Roots和对象路由
  - GC Roots 包括：本地变量、静态变量、JNI 引用等
  - 垃圾收集器内有一组成为 OopMap 的数据结构存储了所有对象的地址
- SafePoint
  - 以“是否具有让程序进入长时间运行的特征”作为标准，如方法跳转、异常跳转
  - GC 过程中用户线程的中断方式是主动式中断，具体行为是用户线程不断轮训收集器的中断标志，如果为真则就近的安全点中断自身
  - 对于已挂起的用户线程，其在挂起之前，先将自身标记为“已进入安全域”（Safe Region），解决用户线程在 Sleep、Blocked 时无法响应 JVM 中断请求而导致 JVM 等待的问题

### 类加载

- 类的生命周期

  <img src="0JavaSummary.assets/image-20210712083625754.png" alt="image-20210712083625754" style="zoom: 33%;" />

- 类的加载顺序

  - 静态变量 > 静态初始化块 > 成员变量 > 初始化块 > 构造器

- 双亲委派模型：

  - 委托给父类加载器加载，防治内存中同一个对象出现多次。
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

- 几个概念、数据结构

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



## 4分布式原理和zookeeper

- CAP
  - 一致性、可用性（**有限时间**内**返回结果**）、分区容错性（部分子网络故障不会导致整个系统不可用）
    - 一致性分为
      - 强一致性
      - 单调一致性 (用户一旦读到某个数据在某次更新后的值，就不会再读到比这个值更旧的值)
      - 会话一致性（TODO)
      - 最终一致性
      - 弱一致性
    - 可用性
      - 从实践的角度来看，即服务是否在任何时刻都可以不超时地响应请求（排除网络超时）

- BASE
  - Basically Available（响应时间增加、功能部分损失如降级）、Soft state（允许不同节点数据副本之间进行数据带来的延迟）、Eventually consistent（数据经过一段时间同步后最终能到达一个一致的状态）

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

  

### Paxos (consensus, not consistency) TODO,而且没总结完

- 三个角色：Proposer、Acceptor、Learner，具体实现中，一个进程可能充当不止一种角色

- 通过不断加强这个约束：“在一次Paxos算法执行实例中，只批准一个value”，获得了 Paxos 算法

  - P1：一个Acceptor必须接受（Accept）它收到的第一个提案

  - P2：一旦一个具有 vn 的提案被批准（chosen），那么之后批准（chosen）的提案必须具有 vn

  - P2a：一旦一个具有 vn 的提案被批准（chosen），那么之后任何acceptor再次接受（accept）的提案必须具有 vn

  - P2b：一旦一个具有 vn 的提案被批准（chosen），那么以后任何proposer提出的提案必须具有 vn

  - P2c

    ：如果一个编号为 n 的提案具有 v ，该提案被提出（issued），那么存在一个多数派S：

    - 要么 S 中所有 Acceptor 都没有接受（accept）编号 < n 的任何提案
    - 要么 S 中所有 Acceptor 已经接受（accept）的编号 < n 的某个提案，而已接收的编号最大的提案的值为 v

  - **P1a**：当且仅当 acceptor 没有回应过编号大于 n 的 prepare 请求时，acceptor 接受（accept）编号为 n 的提案

- proposer 生成提案

  - prepare 阶段
    - proposer 选择一个新的提案编号 n ，然后发送请求给 acceptor 集合，并要求其作如下承诺：
      - acceptor 收到 prepare 请求后，如果提案的编号大于它已经回复过的所有 prepare 消息(**回复消息表示接受 accept**)，则 acceptor 将自己上次接受的提案回复给 proposer，并承诺不再回复小于 n 的提案
    - 举例说明：
      - 假设一个 acceptor 已经响应过（accept）所有的 prepare 请求，对应提案编号为 1、2、3、...、7，那么 acceptor 接收到编号为 8 的 prepare 请求会，就会将编号为 7 的提案作为响应反馈给 proposer
  - accept 阶段
    - 当一个 proposer 收到了多数 acceptors 对 prepare 的回复后，就进入批准（accept）阶段
    - proposer 要向回复 prepare 请求的 acceptors 发送 accept 请求，包括编号 n 和根据 P2c 决定的 value（如果根据P2c没有已经接受的value，那么它可以自由决定value）
    - 只要 acceptor 尚未对编号大于 n 的 prepare 请求响应，就可以通过这个提案

- acceptor 批准提案

  - prepare 请求：acceptor 可以在任何时候响应一个 prepare 请求
  - accept 请求：在不违背 acceptor 现有承诺的前提下，可以任意响应 Accept 请求（一个 acceptor 只要尚未响应任何编号大于 n 的 Prepare 请求，那么它就可以接受这个编号为 n 的提案）

- learner 提案获取（解决单个proposer向大量节点同步数据引起的性能问题）

  - 选取一批 learner 集合作为主 learner 集，acceptor 将批准的提案发送给这个集合，这个集合的每个 learner 可以在一个提案被选定后通知所有其他的 learner

- 算法伪代码

![img](0JavaSummary.assets/120555.jpeg)

### Raft Consensus Algorithm（TODO未总结）

### ZAB（ZooKeeper Atomic Broadcast）

- 定义
  - leader、followers：Zookeeper 集群中，一个节点充当 leader 角色，其余皆为 follower. 
    - leader职责是接受所有客户端请求，协调内部各个服务器。 
    - follower职责是处理客户端非事务请求，参与Proposal的投票和leader选举。
  - transactions：leader 将客户端变更传播给 follower（TODO）
  - 'e'：leader 的纪元 / epoch（任期、term等类似概念）. 纪元是一个整型，leader 当选时的纪元必须比之前的leader纪元要大（TODO）
  - 'c'：一个由 leader 生成的有序数值，从 0 开始并单调递增. 这个值和 epoch 一并用于给接收到的客户端状态变更请求排序（二者通过位运算共同构成 ZXID ）（TODO）
  - 'F.history'：follower 的历史队列，用于提交按顺序接收到的事务（TODO）
  - 待决事务：一组位于 F.history 中的事务，其序列号小于当前提交的序列号（TODO）

- ZAB 所需前提
  1. 复制担保
     - 可靠投递：如果事务 M 被一台服务器提交（commit），其终将被所有服务器提交
     - 全局有序：如果事务 A 于事务 B 之前被一台服务器提交，在所有其他服务器上 A 也将先于B被提交. 如果 A 和 B 都被提交，要么 A 先于 B 被提交，要么 B 先于 A 被提交（不存在同时提交）
     - 因果有序：如果事务 B 在事务 A 提交之后被 sender B 发送，那么 A 的顺序必须在 B 之前. 如果一个sender先发送 B 后发送 C，那么 C 的顺序必须排在 B 之后.（逻辑顺序不受物理条件影响，比如发送端的顺序和接收端最终提交的顺序必须一致）
  2. 只要大多数（quorum）nodes 已启动，事务就会被复制
  3. 若 node 故障但随后又重启，它应能追上故障期间已经复制完成的事务

### 具体实现

- 消息广播复制(2PC的变种)
  - 客户端读取任何一个 Zookeeper 节点
  - 客户端将状态变更写入任何一个 Zookeeper 节点，这个状态变更就会被转发到主（leader）节点
  - Zookeeper 使用一个两阶段提交协议的变种来实现事务复制到 follower
    1. 当 leader 接收到客户端的修改更新请求，它就生成带有序列号 c 和 leader epoch的事务，并发送给所有follower ( leader)
    2. follower 将事务添加到自己的 history queue 中并回复 leader 一个 ACK (follower)
    3. 当 leader 接收到一组 quorum 的 ACK 后，它就向 quorum 发送这个事务的 commit 请求(leader)
    4. follower 会在收到 commit 请求后就提交事务，除非 follower 本地的 history queue 中事务的序列号都低于 c（即历史事务早于当前事务），这时，它将一直等待直到所有早先的事务（待解决事务）的 commit 请求都接收到并处理完成后，才去提交当前的事务 (follower)

<img src="0JavaSummary.assets/120353.png" alt="img" style="zoom:67%;" />

- 集群崩溃恢复

  - 当 leader 崩溃时，所有节点一起执行一个通用的一致恢复协议，之后集群恢复日常运作，确立一个新的 leader 来广播状态变更

  - 要充当 leader 角色，节点必须由一组 quorum nodes 的支持. 由于节点随时可能崩溃和恢复，随着时间推移会产生多个 leader，事实上一个节点可能充当一个角色数次

  - 节点的生命周期：每个节点一次执行协议的一个迭代，任何时候，一个进程可以中断当前迭代，跳转到 Phase 0 重新开始

    - Phase 0：选举（election）
    - Phase 1：发现（discovery）
    - Phase 2：同步（synchronization）
    - Phase 3：广播（broadcast）

  - Phase 1 和 2 对于集群内的相互一致性很重要，尤其是从故障中恢复时

  - Phase 1 发现（摘要：从 quorum 中找到最完备的 F.history）

    - follower 和即将当选的 leader 通信，以便让 leader 收集信息知悉follower最近接收到的事务
    - 这个步骤的目的是发现 quorum 中接收最多的变更序列，并开启新的纪元 epoch = e' 以免先前的leader提交它们的提议（proposal）
    - quorum 拥有先前 leader 发送的所有变更，因此可以做出保证：quorum中至少有一个节点（其 epoch 最大或 epoch 不小于其他跟随者而 lastZxid 最大）的 history queue 中包含先前 leader 发送的所有变更，这同时也意味着新的 leader 也会拥有这些变更

    ![img](../120367.png)

    > 注：理论上被选举出来的 prospective leader 应具有最大的 zxid，即接收了最新的事务，为什么还要向 quorum 中的 follower 获取历史事务？

  - Phase 2 同步（摘要：将'发现'步骤中获得的 F.history 作为提案提出）

    - '同步步骤'使用 leader 的历史更新事务副本来同步，这些事务在'发现步骤'中从集群中获得，同步完成后，整个协议的恢复（recovery）阶段结束 
    - leader 和 follower 通信，将历史事务拿出来作为提案提出（leader）
    - 如果 follower 自身的事务历史序列落后于 leader 的，follower 就认可 leader 的提议（follower）
    - 当 leader 收到 quorum 的确认，它就分发提交（commit）消息至 quorum，此时 leader 就宣称确立，不再是潜在状态（leader）

  ![img](0JavaSummary.assets/120365.png)
  

  - Phase 3 广播
    - 如没有崩溃发生，节点永久地停留在这个阶段，一旦客户端发起一个写请求，节点就执行事务广播（leader）
    - 对于 observer，leader 会发送 inform 消息，其中包含提议的内容（follower）

![img](0JavaSummary.assets/120363.png)

> 为了检测故障，ZAB 在 leader 和 follower 之间使用定期的心跳消息通信. 如果leader 在一段时间内没有收到 quorum（majority）的心跳，它就放弃自己自己的 leader 身份，将状态切换成选举和 Phase 0. 如果 follower 在一段时间内也没收到 leader 的心跳，就跳转到 Leader Election Phase.

- ZXID
  - ZXID 低 32 位为自增计数，高 32 位代表了 Leader 周期 epoch 编号
  - 当选举出一个新的 Leader 时，就会从这个 leader 本地日志中去的最大事务提议的 ZXID，解析出对应的 epoch 值再自加 1，以此作为新的 ZXID 的高 32 位，低 32 位则从 0 开始

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

- **ZAB Leader选举(面试重点：ZAB的领导者选举过程 TODO)**
  
- <img src="0JavaSummary.assets/image-20210630231038099.png" alt="image-20210630231038099" style="zoom:50%;" />
  
  - 服务器状态
    - LOOKING：Leader 选举阶段
    - FOLLOWING：跟随者状态
    - LEADING：领导者状态
    - OBSERVING：观察者状态
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
  
- 数据同步
  - 几个定义
    - peerLastZXID：learner 服务器最后处理的 ZXID
    - miniCommittedLog：leader outstanding proposals queue committedLog 中最小的 ZXID
    - maxCommittedLog：leader outstanding proposals queue committedLog 中最大的 ZXID
  - 直接差异化同步（DIFF同步）：peerLastZXID ∈ (minCommittedLog, maxCommittedLog)
  - 先回滚在差异化同步（TRUNC+DIFF同步），用于Leader宕机时没有成功发起 roposal 但已经将事务记录到本地事务日志中，这时重启服务后，需要先回滚，再做DIFF同步：peerLastZXID ∉ F.history
  - 全量同步（SNAP同步）：peerLastZxid < minCommittedLog || (leader.outstandingProposalsQueue == null && peerLastZXID != lastProcessedZXID)

<img src="0JavaSummary.assets/121186.png" alt="img" style="zoom:120%;" />

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

## 6 kafka

### kafka组件和基本概念

- controller、leader、follower
  - controller负责全局meta信息维护，管理Broker上下线、topic管理、管理分区副本分配、leader选举、管理所有副本状态机和分区状态机；通过zookeeper实现选举
  - leader和follower是针对partition而言，当leader宕机，controller将从ISR中使用分区选择算法选出新的leader
- 

