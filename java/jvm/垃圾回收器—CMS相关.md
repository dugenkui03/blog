[官方链接](https://docs.oracle.com/javase/10/gctuning/concurrent-mark-sweep-cms-collector.htm#JSGCT-GUID-FF8150AC-73D9-4780-91DD-148E63FA1BFF)

##### 1.引言

CMS(concurrent mark sweep)是为那些喜欢喜欢比较短的垃圾回收暂停时间，并且愿意在运行期间与垃圾回收器共享cup资源的应用。

典型的，如果一个应用运行在多核处理器上，并且其老年代拥有比较多的长期存活的数据，那么比较适合使用CMS。CMS可以使用参数`XX:+UseConcMarkSweepGC`开启。

CMS是不宜采用的，强烈建议考虑使用[Garbage-First(G1)](https://docs.oracle.com/javase/10/gctuning/garbage-first-garbage-collector-tuning.htm#JSGCT-GUID-90E30ACA-8040-432E-B3A0-1E0440AB556A)收集器。


##### 2.并发模式失败Concurrent model failure

CMS使用一个或多个与应用线程同时运行的垃圾回收线程来完成垃圾回收的任务。

正如之前所说，CMS在应用线程正在运行的情况下执行大部分的垃圾标记(tracing)和清除工作，所以应用线程看来只会进行短暂的暂停。然而，如果在老年代填满之前CMS不能够完成对不可达对象的回收，或者老年代没有足够多内存完成一次分配请求，然后应用暂停，垃圾回收随着应用线程的停止而完成。‘ then the application is paused and the collection is completed with all the application threads stopped’。并发模式失败指的是不能并发的完成一次收集，这种情况通常需要调整CMS的参数。如果垃圾回收被一次特定的垃圾回收，比如`System.gc()`打断了，或者垃圾回收需要为诊断工具diagnostic tools提供信息，那么并发模式中断会被报告。

##### 3. 过多的GC时间和内存溢出

CMS会抛出`OutOfMemoryError`，如果太多时间消耗在垃圾回收上：如果超过98%的时间消耗在垃圾回收上，而且不到2%的堆内存被回收，那么`OutOfMemoryError`就会被抛出。

这种特性旨在用来保护应用程序长时间运行，同时由于堆内存太小而导致垃圾回收没什么进展。如果需要，这个特性可以使用参数来禁止：`-XX:UseGCOverheadLimit`。

>The policy is the same as that in the parallel collector, except that time spent performing concurrent collections isn't counted toward the 98% time limit. In other words, only collections performed while the application is stopped count toward excessive GC time. Such collections are typically due to a concurrent mode failure or an explicit collection request (for example, a call to System.gc()).

##### 4.CMS收集器和漂浮垃圾Floating Garbage

像java HotSpot中其他垃圾回收器一样，CMS也是一个tracing回收器，即能够识别堆中所有可达的对象。

因为所有的垃圾回收线程和应用线程在老年代回收时并发执行，一个被标记为可达的对象可能在垃圾回收完成之前又变得不可达了——这种不可大的对象就叫做漂浮垃圾floating garbage。漂浮垃圾的数量取决于并发回收持续的时间和对象引用更新的频率。此外，因为年轻代与独立于老年代进行垃圾回收的，而且其每一次垃圾回收都会对后续回收动作产生影响。
建议增加20%的老年代空间来防止过多的漂浮垃圾。

##### 5.CMS暂停pauses

再一次并发垃圾回收周期中，CMS暂停应用线程两次。第一次暂停是为了标记从roots可达的对象。root包括应用线程栈、寄存器、静态对象等。

第一次暂停也称为**初始标记暂停initial mark pause**。第二次暂停是在并发标记阶段concurrent tracing phase，此阶段视为了找到由于线程的引用更新而没有进行标记的对象。
>This first pause is referred to as the initial mark pause. The second pause comes at the end of the concurrent tracing phase and finds objects that were missed by the concurrent tracing due to updates by the application threads of references in an object after the CMS collector had finished tracing that object. This second pause is referred to as the remark pause.

##### 6. CMS收集器并发暂停

可达对象图reachable object graph 的并发追踪发生在初始标记暂停和标记暂停阶段之间。


##### 7.可达性分析

可以作为GC Roots的对象：
1. 方法区中的类静态属性引用的对象——存放在运行时常量池；
2. 方法区中常量引用的对象——final修饰的属性；
3. 虚拟机栈的栈帧中的本地变量表中引用的对象——虚拟机栈为线程独有。


#### 二. CMS执行周期

CMS执行周期分为6个阶段，分别是：`1.初始标记->2.并发标记->3.并发预清理->4.重新标记->5.并发清理->6.重置`。其中1、4初始标记和重新标记阶段都需要暂停所有用户线程。

[注]:GC Root会扫描新生代，即CMS虽然是老年代垃圾回收算法，但也会扫描新生代。

##### 2.1 初始标记

此阶段暂停所有用户线程，并且标记与GC Root**直接关联的对象**。

##### 2.2 并发标记

该阶段进行GC Root tracing：初始标记阶段所有暂停的线程开始运行；从此前标记过的对象出发，标记与GC Root间接关联的对象。

##### 2.3 并发预清理

此阶段用于标记从新生代晋升到老年代的对象、新分配到老年代的大对象和并发标记阶段被修改过的对象。

##### 2.4 重新标记

暂停所有应用线程，重新扫描堆中的对象(新生代、老年代)，进行可达性分析，标记对象。此阶段是多线程的，而且有了前边的基础，所以会很快。

##### 2.5 并发清理

此阶段**清理无效的对象**。

##### 2.6 重置

CMS清除内部状态，为下次回收做准备。



