# JVM 性能调优

回答一些这个几个问题：

- 哪些地方会发生OOM, 调优的思路有哪些？
- 讲讲遇到的OOM的上下文，以及是如何进行分析的，最后怎么解决的？
- FullGC的时候，新生代的存活对象是不是全部会被移动到老年代？不是的，还是按照分代收集。

哪些情况可能会触发 JVM 进行 Full GC。

1. System.gc() 方法的调用此方法的调用是建议 JVM 进行 Full GC，注意这**只是建议而非一定**，但在很多情况下它会触发 Full GC，从而增加 Full GC 的频率。通常情况下我们只需要让虚拟机自己去管理内存即可，我们可以通过 -XX:+ DisableExplicitGC 来禁止调用 System.gc()。

2. 老年代空间不足老年代空间不足会触发 Full GC操作，若进行该操作后空间依然不足，则会抛出如下错误：<br>
   ` java.lang.OutOfMemoryError: Java heap space `

3. 永久代空间不足
   JVM 规范中运行时数据区域中的方法区，在 HotSpot 虚拟机中也称为永久代（Permanet Generation），存放一些类信息、常量、静态变量等数据，当系统要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，会触发 Full GC。如果经过 Full GC 仍然回收不了，那么 JVM 会抛出如下错误信息：<br>
   `java.lang.OutOfMemoryError: PermGen space `

4. CMS GC 时出现 promotion failed 和 concurrent mode failurepromotion failed，就是上文所说的担保失败，而 concurrent mode failure 是在执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足造成的。

5. 统计得到的Minor GC晋升到旧生代的平均大小大于老年代的剩余空间
