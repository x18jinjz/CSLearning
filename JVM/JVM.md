[TOC]



## 一、内存结构

JVM内存结构主要有三大块：**堆内存**、**栈**和 **方法区**。 

控制参数：

- -Xms设置堆的最小空间大小。
- -Xmx设置堆的最大空间大小。
- -XX:NewSize设置新生代最小空间大小。
- -XX:MaxNewSize设置新生代最大空间大小。
- -XX:PermSize设置永久代最小空间大小。
- -XX:MaxPermSize设置永久代最大空间大小。
- -Xss设置每个线程的堆栈大小。
- 没有直接设置老年代的参数 。老年代空间大小=堆空间大小-年轻代大空间大小 

![img](https://mmbiz.qpic.cn/mmbiz_png/PgqYrEEtEnoUSbbnzEiafyyQWUibOfnE3GicpdRQOuxWBrhB3Fic7MRf4z5ywT2RmCicibGibHNQEgUbsibLR1eLVRfo3A/640?wx_fmt=png&wxfrom=5&wx_lazy=1) 

![img](https://mmbiz.qpic.cn/mmbiz_png/PgqYrEEtEnoUSbbnzEiafyyQWUibOfnE3G1sZG1aJZSakhFe5d6QeiciaO9ZIDfHrFS9UZx8RfWfkPk9UZLCVdcriaQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1) 



### 堆

堆内存是JVM中最大的一块，由所有线程共享，在虚拟机启动时创建。唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。 

可以分为**年轻代**和**老年代**。年轻代又分为**Eden空间**、**From Survivor空间**、**To Survivor空间** ，默认情况是8:1:1。

可以处于物理上不连续空间，只要逻辑上连续。如果堆中没有内存完成实例分配，并且无法再扩展时，会抛出`OutOfMemoryError`异常。

### 方法区

也称为Non-Heap（非堆），目的是与Java堆区分开来，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。 有时被称为持久代（PermGen） 。

当方法区无法满足内存分配需求时，将抛出`OutOfMemoryError`异常。 

### 程序计数器

程序计数器（Program  Counter  Register）是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过一些更高效的方式去实现），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是Natvie方法，这个计数器值则为空（Undefined）。

此内存区域是**唯一一个在Java虚拟机规范中没有规定任何`OutOfMemoryError`情况的区域。**

### JVM栈

与程序计数器一样，Java虚拟机栈也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：**每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。**每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。**

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

在Java虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出`StackOverflowError`异常；如果虚拟机栈可以动态扩展，当扩展时无法申请到足够的内存时会抛出`OutOfMemoryError`异常。

### 本地方法栈

本地方法栈与虚拟机栈所发挥的作用是非常相似的，其区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而**本地方法栈则是为虚拟机使用到的Native方法服务。**与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。



## 二、GC

### 对象存活判断

1.  **引用计数算法**：每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。此方法简单，无法解决对象相互循环引用的问题。 

2.  **可达性分析**：从GC Roots开始向下搜索，搜索所走过的路径称为引用链。当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的，即不可达对象。 

   GC Roots包括：

   ​	虚拟机栈中引用的对象。

   ​	方法区中类静态属性实体引用的对象。

   ​	方法区中常量引用的对象。

   ​	本地方法栈中JNI引用的对象。

### finalize（）自救

当一个对象不可达时，也不一定会被回收，需要经过两次标记过程。第一次标记时会判断是否需要执行`finalize`方法。如果没有覆盖finalize方法或者finalize已经被虚拟机调用过（只会执行一次），则为没有必要执行，回收。如果需要执行，被放置到一个队列中，随后会去调用finalize方法，对象可能在finalize方法中重新与引用链上的对象建立关联，譬如把自己（this）赋值给某个类或其他对象的成员变量，则第二次标记时被移除出“即将回收”的集合，如果执行finalize之后仍没有逃脱，则被回收。

### GC算法

1、标记 - 清除算法

将需要回收的对象进行标记，然后清除。

不足：

1. 标记和清除过程效率都不高；
2. 会产生大量碎片，内存碎片过多可能导致无法给大对象分配内存。

 ![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/a4248c4b-6c1d-4fb8-a557-86da92d3a294.jpg)

2、复制算法

将内存划分为大小相等的两块，每次只使用其中一块，当这一块内存用完了就将还存活的对象复制到另一块上面，然后再把使用过的内存空间进行一次清理。

不足：只使用了内存的一半。

大多采用这种收集算法来回收新生代，但是并不是将内存划分为大小相等的两块，而是分为一块较大的 Eden 空间和两块较小的  Survior 空间，每次使用 Eden 空间和其中一块 Survivor。在回收时，将 Eden 和 Survivor  中还存活着的对象一次性复制到另一块 Survivor 空间上，最后清理 Eden 和 使用过的那一块 Survivor。HotSpot 虚拟机的 Eden 和 Survivor 的大小比例默认为 8:1，保证了内存的利用率达到 90 %。如果每次回收有多于 10% 的对象存活，那么一块  Survivor 空间就不够用了，此时需要依赖于老年代进行分配担保，也就是借用老年代的空间。 

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/e6b733ad-606d-4028-b3e8-83c3a73a3797.jpg) 

3、标记 - 整理算法

复制收集算法在对象存活率较高时就要执行较多的复制操作，效率将会变低，还需要有额外的空间进行分配担保，所以在老年代一般不能直接选用这种算法。

“标记-整理”算法的标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/902b83ab-8054-4bd2-898f-9a4a0fe52830.jpg) 

4、分代收集算法

现在的商业虚拟机采用分代收集算法，它根据对象存活周期将内存划分为几块，不同块采用适当的收集算法。

一般将 Java 堆分为新生代和老年代。

1. 新生代使用：复制算法
2. 老年代使用：标记 - 清除 或者 标记 - 整理 算法

### 垃圾回收器

1、Serial收集器

 [![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/22fda4ae-4dd5-489d-ab10-9ebfdad22ae0.jpg)](https://github.com/CyC2018/Interview-Notebook/blob/master/pics/22fda4ae-4dd5-489d-ab10-9ebfdad22ae0.jpg) 

 

它是单线程的收集器，不仅意味着只会使用一个线程进行垃圾收集工作，更重要的是它在进行垃圾收集时，必须暂停所有其他工作线程（Stop The World），往往造成过长的等待时间。

它的优点是简单高效，对于单个 CPU 环境来说，由于没有线程交互的开销，因此拥有最高的单线程收集效率。

在 Client 应用场景中，分配给虚拟机管理的内存一般来说不会很大，该收集器收集几十兆甚至一两百兆的新生代停顿时间可以控制在一百多毫秒以内，只要不是太频繁，这点停顿是可以接受的。

2、ParNew收集器

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/81538cd5-1bcf-4e31-86e5-e198df1e013b.jpg) 

其实是是 Serial 收集器的多线程版本，Server 模式下的虚拟机首选新生代收集器，除了性能原因外，主要是因为除了 Serial 收集器，只有它能与 CMS 收集器配合工作。 

3、Parallel Scavenge 收集器

新生代复制算法、老年代标记-压缩 

与ParNew类似，目标是达到一个可控制的吞吐量，它被称为“吞吐量优先”收集器。这里的吞吐量指 CPU 用于运行用户代码的时间占总时间的比值。 

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验。而高吞吐量则可以高效率地利用 CPU 时间，尽快完成程序的运算任务，主要适合在后台运算而不需要太多交互的任务。 

可以通过参数来打开自适应调节策略，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或最大的吞吐量（自适应调节策略也是它与 ParNew 收集器的一个重要区别 ）；也可以通过参数控制GC的时间不大于多少毫秒或者比例。

4、Serial Old收集器

 [![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/08f32fd3-f736-4a67-81ca-295b2a7972f2.jpg)](https://github.com/CyC2018/Interview-Notebook/blob/master/pics/08f32fd3-f736-4a67-81ca-295b2a7972f2.jpg) 

Serial Old 是 Serial 收集器的老年代版本，也是给 Client 模式下的虚拟机使用。如果用在 Server 模式下，它有两大用途：

1. 在 JDK 1.5 以及之前版本（Parallel Old 诞生以前）中与 Parallel Scavenge 收集器搭配使用。
2. 作为 CMS 收集器的后备预案，在并发收集发生 Concurrent Mode Failure 时使用。

5、Parallel Old收集器

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/278fe431-af88-4a95-a895-9c3b80117de3.jpg) 

是 Parallel Scavenge 收集器的老年代版本。在注重吞吐量以及 CPU 资源敏感的场合，都可以优先考虑 Parallel Scavenge 加 Parallel Old 收集器。

6、CMS（Concurrent Mark Sweep） 收集器

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/62e77997-6957-4b68-8d12-bfd609bb2c68.jpg) 

以获取最短停顿时间为目标，采用标记-清除算法，并发收集，低停顿。

整个GC过程分为四个步骤：

1. 初始标记：仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快，需要停顿。
2. 并发标记：进行 GC Roots Tracing 的过程，它在整个回收过程中耗时最长，不需要停顿。
3. 重新标记：为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要停顿。
4. 并发清除：不需要停顿。

**初始标记**和**重新标记**需要停顿。耗时最长的**并发标记**和**并发清除**过程中，收集器线程都可以与用户线程一起工作，不需要进行停顿。 

缺点：

1. 对 CPU 资源敏感。CMS 默认启动的回收线程数是 (CPU 数量 + 3) / 4，当 CPU 不足 4 个时，CMS  对用户程序的影响就可能变得很大，如果本来 CPU 负载就比较大，还要分出一半的运算能力去执行收集器线程，就可能导致用户程序的执行速度忽然降低了  50%，其实也让人无法接受。并且低停顿时间是以牺牲吞吐量为代价的，导致 CPU 利用率变低。
2. 无法处理浮动垃圾。由于并发清理阶段用户线程还在运行着，伴随程序运行自然就还会有新的垃圾不断产生。这一部分垃圾出现在标记过程之后，CMS  无法在当次收集中处理掉它们，只好留到下一次 GC  时再清理掉，这一部分垃圾就被称为“浮动垃圾”。也是由于在垃圾收集阶段用户线程还需要运行，那也就还需要预留有足够的内存空间给用户线程使用，因此它不能像其他收集器那样等到老年代几乎完全被填满了再进行收集，需要预留一部分空间提供并发收集时的程序运作使用。可以使用  -XX:CMSInitiatingOccupancyFraction 的值来改变触发收集器工作的内存占用百分比，JDK 1.5  默认设置下该值为 68，也就是当老年代使用了 68% 的空间之后会触发收集器工作。如果该值设置的太高，导致浮动垃圾无法保存，那么就会出现  Concurrent Mode Failure，此时虚拟机将启动后备预案：临时启用 Serial Old 收集器来重新进行老年代的垃圾收集。
3. 标记 - 清除算法导致的空间碎片，给大对象分配带来很大麻烦，往往出现老年代空间剩余，但无法找到足够大连续空间来分配当前对象，不得不提前触发一次 Full GC。

7、G1收集器

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/f99ee771-c56f-47fb-9148-c0036695b5fe.jpg) 

具备如下特点：

- 并行与并发：能充分利用多 CPU 环境下的硬件优势，使用多个 CPU 来缩短停顿时间。
- 分代收集：分代概念依然得以保留，虽然它不需要其它收集器配合就能独立管理整个 GC 堆，但它能够采用不同方式去处理新创建的对象和已存活一段时间、熬过多次 GC 的旧对象来获取更好的收集效果。
- 空间整合：整体来看是基于“标记 - 整理”算法实现的收集器，从局部（两个 Region 之间）上来看是基于“复制”算法实现的，这意味着运行期间不会产生内存空间碎片。
- 可预测的停顿：这是它相对 CMS 的一大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1  除了降低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为 M 毫秒的时间片段内，消耗在 GC 上的时间不得超过 N  毫秒，这几乎已经是实时 Java（RTSJ）的垃圾收集器的特征了。

在 G1 之前的其他收集器进行收集的范围都是整个新生代或者老生代，而 G1 不再是这样，它将整个 Java  堆划分为多个大小相等的独立区域（Region）。虽然还保留新生代和老年代的概念，但新生代和老年代不再是物理隔离的了，而都是一部分  Region（不需要连续）的集合。

之所以能建立可预测的停顿时间模型，是因为它可以有计划地避免在整个 Java 堆中进行全区域的垃圾收集。它跟踪各个 Region  里面的垃圾堆积的价值大小（回收所获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的  Region（这也就是 Garbage-First 名称的来由）。这种使用 Region  划分内存空间以及有优先级的区域回收方式，保证了它在有限的时间内可以获取尽可能高的收集效率。

不能保证GC真的以Region为单位进行。Region 不可能是孤立的，一个对象分配在某个 Region 中，可以与整个 Java  堆任意的对象发生引用关系。在做可达性分析确定对象是否存活的时候，需要扫描整个 Java 堆才能保证准确性，这显然是对 GC  效率的极大伤害。为了避免全堆扫描的发生，每个 Region 都维护了一个与之对应的 Remembered Set。虚拟机发现程序在对  Reference 类型的数据进行写操作时，会产生一个 Write Barrier 暂时中断写操作，检查 Reference  引用的对象是否处于不同的 Region 之中，如果是，便通过 CardTable 把相关引用信息记录到被引用对象所属的 Region 的  Remembered Set 之中。当进行内存回收时，在 GC 根节点的枚举范围中加入 Remembered Set  即可保证不对全堆扫描也不会有遗漏。

如果不计算维护 Remembered Set 的操作，G1 收集器的运作大致可划分为以下几个步骤：

1. 初始标记
2. 并发标记
3. 最终标记：为了修正在并发标记期间因用户程序继续运作而导致标记产生变动的那一部分标记记录，虚拟机将这段时间对象变化记录在线程的  Remembered Set Logs 里面，最终标记阶段需要把 Remembered Set Logs 的数据合并到 Remembered  Set 中。这阶段需要停顿线程，但是可并行执行。
4. 筛选回收：首先对各个 Region 中的回收价值和成本进行排序，根据用户所期望的 GC 停顿是时间来制定回收计划。此阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分 Region，时间是用户可控制的，而且停顿用户线程将大幅度提高收集效率。

**七种回收器比较**

|        收集器         | 串行、并行 or 并发 | 新生代 / 老年代 |         算法         |     目标     |                   适用场景                    |
| :-------------------: | :----------------: | :-------------: | :------------------: | :----------: | :-------------------------------------------: |
|      **Serial**       |        串行        |     新生代      |       复制算法       | 响应速度优先 |          单 CPU 环境下的 Client 模式          |
|      **ParNew**       |        并行        |     新生代      |       复制算法       | 响应速度优先 |   多 CPU 环境时在 Server 模式下与 CMS 配合    |
| **Parallel Scavenge** |        并行        |     新生代      |       复制算法       |  吞吐量优先  |       在后台运算而不需要太多交互的任务        |
|    **Serial Old**     |        串行        |     老年代      |      标记-整理       | 响应速度优先 |  单 CPU 环境下的 Client 模式、CMS 的后备预案  |
|   **Parallel Old**    |        并行        |     老年代      |      标记-整理       |  吞吐量优先  |       在后台运算而不需要太多交互的任务        |
|        **CMS**        |        并发        |     老年代      |      标记-清除       | 响应速度优先 | 集中在互联网站或 B/S 系统服务端上的 Java 应用 |
|        **G1**         |        并发        |      both       | 标记-整理 + 复制算法 | 响应速度优先 |         面向服务端应用，将来替换 CMS          |



### 内存分配和回收机制

关于 Minor GC 和 Full GC：

- Minor GC：发生在新生代上，因为新生代对象存活时间很短，因此 Minor GC 会频繁执行，执行的速度一般也会比较快。
- Full GC：发生在老年代上，老年代对象和新生代的相反，其存活时间长，因此 Full GC 很少执行，而且执行速度会比 Minor GC 慢很多。

对象的内存分配，也就是在堆上分配。主要分配在新生代的 Eden 区上，少数情况下也可能直接分配在老年代中。 主要的分配策略有：

1. **优先分配在Eden区**

   大多数情况下，对象在新生代 Eden 区分配，当 Eden 区空间不够时，发起 Minor GC。 

2. **大对象直接进入老年代**

   大对象是指需要连续内存空间的对象，最典型的大对象是那种很长的字符串以及数组。经常出现大对象会提前触发垃圾收集以获取足够的连续空间分配给大对象。 

3. **长期存活对象进入老年代**

   JVM 为对象定义年龄计数器，经过 Minor GC 依然存活，并且能被 Survivor 区容纳的，移被移到 Survivor 区，年龄就增加 1 岁，增加到一定年龄则移动到老年代中（默认 15 岁，通过 -XX:MaxTenuringThreshold 设置）。 

4. 动态对象年龄判断

   JVM 并不是永远地要求对象的年龄必须达到 MaxTenuringThreshold 才能晋升老年代，如果在 **Survivor  区中相同年龄所有对象大小的总和大于 Survivor 空间的一半，则年龄大于或等于该年龄的对象可以直接进入老年代**，无需等待  MaxTenuringThreshold 中要求的年龄。 

5. 空间分配担保

   在发生 Minor GC 之前，JVM 先检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果条件成立的话，那么 Minor GC  可以确认是安全的；如果不成立的话 JVM 会查看 HandlePromotionFailure  设置值是否允许担保失败，如果允许那么就会继续检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次  Minor GC，尽管这次 Minor GC 是有风险的；如果小于，或者 HandlePromotionFailure  设置不允许冒险，那这时也要改为进行一次 Full GC。 

### Full GC 的触发条件

对于 Minor GC，其触发条件非常简单，当 Eden 区空间满时，就将触发一次 Minor GC。而 Full GC 则相对复杂，有以下条件：

1. 调用System.gc()

   此方法的调用是建议 JVM 进行 Full GC，虽然只是建议而非一定，但很多情况下它会触发 Full GC，从而增加 Full GC  的频率，也即增加了间歇性停顿的次数。因此强烈建议能不使用此方法就不要使用，让虚拟机自己去管理它的内存。可通过 -XX:+  DisableExplicitGC 来禁止 RMI 调用 System.gc()。

2. 老年代空间不足

   老年代空间不足的常见场景为前文所讲的大对象直接进入老年代、长期存活的对象进入老年代等，当执行 Full GC 后空间仍然不足，则抛出  Java.lang.OutOfMemoryError。为避免以上原因引起的 Full GC，调优时应尽量做到让对象在 Minor GC  阶段被回收、让对象在新生代多存活一段时间以及不要创建过大的对象及数组。

3. 空间分配担保失败

   使用复制算法的 Minor GC 需要老年代的内存空间作担保，如果出现了 HandlePromotionFailure 担保失败，则会触发 Full GC。

4. JDK 1.7 及以前的永久代空间不足

   在 JDK 1.7 及以前，HotSpot 虚拟机中的方法区是用永久代实现的，永久代中存放的为一些 Class  的信息、常量、静态变量等数据，当系统中要加载的类、反射的类和调用的方法较多时，永久代可能会被占满，在未配置为采用 CMS GC 的情况下也会执行  Full GC。如果经过 Full GC 仍然回收不了，那么 JVM 会抛出  `OutOfMemoryError`，为避免以上原因引起的 Full GC，可采用的方法为增大永久代空间或转为使用 CMS  GC。

5. Concurrent Mode Failure

   执行 CMS GC 的过程中同时有对象要放入老年代，而此时老年代空间不足（有时候“空间不足”是 CMS GC  时当前的浮动垃圾过多导致暂时性的空间不足触发 Full GC），便会报 Concurrent Mode Failure 错误，并触发 Full  GC。



## 三、类加载机制

### 1.类的加载

类的加载指的是将类的.class文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个 `java.lang.Class`对象，用来封装类在方法区内的数据结构。类的加载的最终产品是位于堆区中的 `Class`对象， `Class`对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。 

不需要等到某个类被“首次主动使用”时再加载它，JVM规范允许类加载器在预料某个类将要被使用时就预先加载它，如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序首次主动使用该类时才报告错误（LinkageError错误），如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误。

### 2.类的生命周期

![img](https://github.com/CyC2018/Interview-Notebook/raw/master/pics/32b8374a-e822-4720-af0b-c0f485095ea2.jpg) 



包括以下 7 个阶段：

- **加载（Loading）**
- 连接
  - **验证（Verification）**
  - **准备（Preparation）**
  - **解析（Resolution）**
- **初始化（Initialization）**
- 使用（Using）
- 卸载（Unloading）

其中解析过程在某些情况下可以在初始化阶段之后再开始，这是为了支持 Java 的动态绑定。

**加载**

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时存储结构。
3. 在内存中生成一个代表这个类的 Class 对象，作为方法区这个类的各种数据的访问入口。

**验证：确保被加载的类的正确性**

验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。验证阶段大致会完成4个阶段的检验动作：

- **文件格式验证**：验证字节流是否符合Class文件格式的规范；例如：是否以 `0xCAFEBABE`开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
- **元数据验证**：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了 `java.lang.Object`之外。
- **字节码验证**：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
- **符号引用验证**：确保解析动作能正确执行。

验证阶段是非常重要的，但不是必须的，它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用 `-Xverifynone`参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

**准备**：为类的**静态变量**分配内存，并将其初始化为默认值

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中分配。对于该阶段有以下几点需要注意：

- 1、这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
- 2、这里所设置的初始值通常情况下是数据类型默认的零值（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。
- 3、如果类字段的字段属性表中存在 `ConstantValue`属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。 

假设一个类变量的定义为： `public static int value = 3`；那么变量value在准备阶段过后的初始值为0，而不是3，因为这时候尚未开始执行任何Java方法，而把value赋值为3的`public static`指令是在程序编译后，存放于类构造器 `<clinit>（）`方法之中的，所以把value赋值为3的动作将在初始化阶段才会执行。 

假设上面的类变量value被定义为： `public static final int value = 3`；编译时Javac将会为value生成ConstantValue属性，在准备阶段虚拟机就会根据 `ConstantValue`的设置将value赋值为3。我们可以理解为static final常量在编译期就将其结果放入了调用它的类的常量池中。

> 这里还需要注意如下几点：
>
> - 对基本数据类型来说，对于静态类变量（static）和全局变量，如果不显式地对其赋值而直接使用，则系统会为其赋予默认的零值，而对于局部变量来说，在使用前必须显式地为其赋值，否则编译时不通过。
> - 对于同时被static和final修饰的常量，必须在声明的时候就为其显式地赋值，否则编译时不通过；而只被final修饰的常量则既可以在声明时显式地为其赋值，也可以在类初始化时显式地为其赋值，总之，在使用前必须为其显式地赋值，系统不会为其赋予默认零值。
> - 对于引用数据类型reference来说，如数组引用、对象引用等，如果没有对其进行显式地赋值而直接使用，系统都会为其赋予默认的零值，即null。
> - 如果在数组初始化时没有对数组中的各元素赋值，那么其中的元素将根据对应的数据类型而被赋予默认的零值。

**解析：把类中的符号引用转换为直接引用**

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。

符号引用就是一组符号来描述目标，可以是任何字面量。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

**初始化**

为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化。在Java中对类变量进行初始值设定有两种方式：

- ①声明类变量时指定初始值
- ②使用静态代码块为类变量指定初始值

JVM初始化步骤

- 1、假如这个类还没有被加载和连接，则程序先加载并连接该类
- 2、假如该类的直接父类还没有被初始化，则先初始化其直接父类
- 3、假如类中有初始化语句，则系统依次执行这些初始化语句

类初始化时机：只有当对类的主动使用的时候才会导致类的初始化，类的主动使用包括以下六种：

- 创建类的实例，也就是new的方式
- 访问某个类或接口的静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 反射（如 `Class.forName(“com.shengsiyuan.Test”)`）
- 初始化某个类的子类，则其父类也会被初始化
- Java虚拟机启动时被标明为启动类的类（ `JavaTest`），直接使用 `java.exe`命令来运行某个主类

**结束生命周期**

在如下几种情况下，Java虚拟机将结束生命周期

- 执行了 `System.exit()`方法
- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而异常终止
- 由于操作系统出现错误而导致Java虚拟机进程终止

### 3.类加载器

![img](http://mmbiz.qpic.cn/mmbiz_png/PgqYrEEtEnokxXiapDdvntH8PGwa0zGXM7qXKib1ibsib1BuyLxjoP1sgorwib78yTD4896N5r1AibdhDXTHZ7z9VyBg/640?wx_fmt=png&wxfrom=5&wx_lazy=1) 

从 Java 虚拟机的角度来讲，只存在以下两种不同的类加载器：

- 启动类加载器（Bootstrap ClassLoader），这个类加载器用 C++ 实现，是虚拟机自身的一部分；
- 所有其他类的加载器，这些类由 Java 实现，独立于虚拟机外部，并且全都继承自抽象类 java.lang.ClassLoader。

从 Java 开发人员的角度看，类加载器可以划分得更细致一些：

- 启动类加载器（Bootstrap ClassLoader）此类加载器负责将存放在 <JAVA_HOME>\lib  目录中的，或者被 -Xbootclasspath 参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如  rt.jar，名字不符合的类库即使放在 lib 目录中也不会被加载）类库加载到虚拟机内存中。 启动类加载器无法被 Java  程序直接引用。
- 扩展类加载器（Extension ClassLoader）这个类加载器是由  ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将  <JAVA_HOME>/lib/ext 或者被 java.ext.dir  系统变量所指定路径中的所有类库加载到内存中，开发者可以直接使用扩展类加载器。
- 应用程序类加载器（Application ClassLoader）这个类加载器是由  AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的，一般称为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

**JVM类加载机制**

- **全盘负责**，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入。
- **父类委托**，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。
- **缓存机制**，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效。

### 4、类的加载

类加载有三种方式：

- 1、命令行启动应用时候由JVM初始化加载
- 2、通过Class.forName()方法动态加载
- 3、通过ClassLoader.loadClass()方法动态加载

`Class.forName()`：将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块；

`ClassLoader.loadClass()`：只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。

#### 5、双亲委派模型

**工作流程**：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

## 5、双亲委派模型

双亲委派模型的工作流程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

这里类加载器之间的父子关系一般通过组合（Composition）关系来实现，而不是通过继承（Inheritance）的关系实现。 

**好处**：系统类防止内存中出现多份同样的字节码，保证Java程序安全稳定运行 。