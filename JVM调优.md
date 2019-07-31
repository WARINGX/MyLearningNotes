---
title: JVM调优
date: 2019-07-26 13:49:18
tags:
---

# JVM参数设置

![img](https://img-blog.csdn.net/2018041715401852)

-Xms设置堆的最小空间大小。
堆中 年轻代和年老有个默认比，比如 是  NewRatio   = 2 (默认是 2:1)
年轻代中eden和suvivor有个默认比例 8:1:1  （SurvivorRatio = 8）  jps查看进程 jmap -heap 进程编号 查看到改参数
-Xmx设置堆的最大空间大小。

-XX:NewSize设置新生代最小空间大小。

-XX:MaxNewSize设置新生代最大空间大小。

-XX:PermSize设置永久代最小空间大小。

-XX:MaxPermSize设置永久代最大空间大小。

-Xss设置每个线程的堆栈大小 （64位 默认是1M  -XX:ThreadStackSize默认是0）

# 垃圾回收

JVM 中，程序计数器、Java栈、本地方法栈都是随线程而生随线程而灭，栈帧随着方法的进入和退出做入栈和出栈操作，实现了自动的内存清理，因此，我们的内存垃圾回收主要集中于 Java 堆和方法区中，在程序运行期间，这部分内存的分配和使用都是动态的.

#### 对象存活判断

引用计数：每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。此方法简单，无法解决对象相互循环引用的问题。 

可达性分析（Reachability Analysis）：从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的，不可达对象。

#### 垃圾回收算法

判断对象是否存活后  需要对这些对象进行标记 然后从对应区删除 常用算法 如下：

##### 标记 -清除算法（Mark-Sweep）

算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收掉所有被标记的对象。之所以说它是最基础的收集算法，是因为后续的收集算法都是基于这种思路并对其缺点进行改进而得到的。

![img](https://img-blog.csdn.net/20180417164348571)

##### 复制算法（Copying）

复制（Copying）的收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。如果对象大部分都是存活的  移动效率低下 只适用于大部分需要被删除少部分存活 少部分移动的情况

![img](https://img-blog.csdn.net/20180417164357732)

##### 标记-压缩（整理）算法（Mark-Compact）

标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存

![img](https://img-blog.csdn.net/20180417164951325)

##### 分代收集算法

GC分代的基本假设：绝大部分对象的生命周期都非常短暂，存活时间短。

“分代收集”（Generational Collection）算法，把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或“标记-整理”算法来进行回收。

#### 垃圾收集器

##### 垃圾收集算法是内存回收的方法论，垃圾收集器就是内存回收的具体实现 

以下使用的-XX参数语法是
  1) 布尔型参数选项：-XX:+ 打开， -XX:- 关闭。（译者注：比如-XX:+PrintGCDetails）
  2) 数字和字符型型参数选项通过-XX:=设定。数字可以是 m/M(兆字节)，k/K(千字节)，g/G(G字节)。比如：32K表示32768字节。（比如-X**X:HeapDumpPath=./java_pid.hprof）**

#### **Serial收集器**

串行收集器是**最古老**，**最稳定**以及**效率高**的收集器，可能会**产生较长的停顿**，只使用**一个线程**去回收。**新生代**、**老年代**使用串行回收；**新生代复制算法、老年代标记-压缩**；垃圾收集的过程中会Stop The World（服务暂停） 该收集器包括：

##### **串行收集器(新老年代都是单线程)**: 参数控制：-XX:+UseSerialGC 

##### **ParNew收集器（新生代多线程 老年代单线程）**:

#####  ParNew收集器其实就是Serial收集器的多线程版本。新生代并行，老年代串行；**新生代复制算法、老年代标记-压缩**

参数控制：
    `-XX:+UseParNewGC ParNew收集器-XX:ParallelGCThreads 限制线程数量 默认是cpu核数 通过以下命令查看 （java -XX:+PrintFlagsFinal -version | find "ParallelGCThreads" ）C:\Users\jiaozi>java -XX:+PrintFlagsFinal -version | find "ParallelGCThreads"uintx ParallelGCThreads                         = 4{product}`

### Parallel收集器

##### **Parallel Scavenge收集器 类似ParNew收集器**

Parallel收集器更关注系统的吞吐量。可以通过**参数**来打开**自适应调节策略**，虚拟机会根据当前系统的运行情况**收集性能监控信息**，**动态调整**这些**参数**以提供**最合适的停顿时间**或**最大的吞吐量**；也可以通过参数控制GC的时间不大于多少毫秒或者比例；新生代复制算法、老年代**标记-压缩**

参数控制：`-XX:+UseParallelGC 使用Parallel收集器+ 老年代串行 查看该参数 默认值 java -XX:+PrintFlagsFinal -version | find "UseParallelGC"   默认开启`
Parallel Old 收集器

##### **Parallel Old是Parallel Scavenge收集器的老年代版本**，使用多线程和**“标记－整理”**算法。

这个收集器是在JDK 1.6中才开始提供
参数控制： -XX:+UseParallelOldGC 使用Parallel收集器+ 老年代并行 同上查看 默认开启

**CMS收集器**

​        CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。目前很大一部分

的Java应用都集中在互联网站或B/S系统的服务端上，这类应用尤其重视服务的响应速度，希望系统停顿时间最

短，以给用户带来较好的体验。从名字（包含“Mark Sweep”）上就可以看出CMS收集器是基于**“标记-清除”**算法实

现的
    **优点: 并发收集、低停顿** 
    **缺点: 产生大量空间碎片、并发阶段会降低吞吐量**
参数控制：

`-XX:+UseConcMarkSweepGC 使用CMS收集器`
`-XX:+ UseCMSCompactAtFullCollection Full GC后，进行一次碎片整理；整理过程是独占的，会引起停顿时间变长`
`-XX:+CMSFullGCsBeforeCompaction 设置进行几次Full GC后，进行一次碎片整理`
`-XX:ParallelCMSThreads 设定CMS的线程数量（一般情况约等于可用CPU数量）`

**G1收集器**

​     G1是目前技术发展的最前沿成果之一，HotSpot开发团队赋予它的使命是未来可以替换掉JDK1.5中发布的CMS

收集器。与CMS收集器相比G1收集器有以下特点：

空间整合，G1收集器采用**标记整理算法**，**不会产生内存空间碎片**。分配大对象时不会因为无法找到连续空间而提

前触发下一次GC。

**可预测停顿**，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立

**可预测的停顿时间模型**，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不

得超过N毫秒，这几乎已经是实时Java（RTSJ）的垃圾收集器的特征了。

参数配置：

`-XX:+UnlockExperimentalVMOptions -XX:+UseG1GC #开启；`

`-XX:MaxGCPauseMillis =50 #暂停时间目标；`

`-XX:GCPauseIntervalMillis =200 #暂停间隔目标；`

`-XX:+G1YoungGenSize=512m #年轻代大小；`

`-XX:SurvivorRatio=6 #幸存区比例`

### 垃圾回收如何执行

当年轻代内存满时，会引发一次普通GC，该GC仅回收年轻代。需要强调的时，年轻代满是指Eden代满，Survivor满不会引发GC
当年老代满时会引发Full GC，Full GC将会同时回收年轻代、年老代
当永久代满时也会引发Full GC，会导致Class、Method元信息的卸载

### 何时会抛出OutOfMemoryException

并不是内存被耗空的时候才抛出，JVM98%的时间都花费在内存回收，每次回收的内存小于2%
满足这两个条件将触发OutOfMemoryException，这将会留给系统一个微小的间隙以做一些Down之前的操作，比如手动打印Heap Dump。

























