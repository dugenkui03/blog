
##### 引言

内存可粗分为**堆**和**栈**。

堆又可以分为年轻代和永久代；

栈可以分为永久带(class信息、常量和静态变量)、虚拟机栈和本地方法栈。



##### 1. `System.gc()`的调用

此方法是建议JVM进行Full gc，一般会触发Full gc。

`-XX:+DisableExplicitGC`可禁止代码中显式的GC调用。建议加上此配置禁用此方法，由jvm自己控制gc。


<font color=red>
符合以下几种条件则可能：

1. 新生代、老年代和永久代占用少、增长缓慢；
2. 没有使用`-XX:+DisableExplicitGC`；
3. 代码中没有大对象和数组的创建。

</font>

<font color=blue>


</font>

##### 2.老年代空间不足`-XX:+CMSInitiatingOccupancyFraction`

CMS可以使用 **`-XX:+CMSInitiatingOccupancyFraction`来设置老年代空间使用多少比例** 后触发full gc。老年代空间不够用情况一般如下：

<font color=red>
符合以下集中条件则可能：

1. 新生代对象转到老年代对象时老年代空间不足。新生代对象转到老年代条件如下：
    - 新生代对象超过了`PertenureSizeThreshold`设置的大小。比如大对象list和数组；
    - 新生代经过的回收次数达到了`MaxTenureThreshould`设置的次数；
    - minor gc时survivor space放不下存活的对象，对象只能放进老年代。
2. 老年代空间够，但是晋升的是大对象，没有连续的内存空间存放对象。

</font>

##### 3.统计得到的以往晋升到老年代的对象平均大小大于老年代当前剩余空间

当进行 **minor gc**时(达到年龄的对象需要放到老年代)，如果老年代剩余空间小于<font color=red>以往晋升到老年代的所有对象平均大小大</font>，则进行full gc。

##### 4. 永久代空间不足

永久带存放 **class信息、常量(final)、静态变量(static)等**，当系统中加载的类、发射调用的类和**调用的方法**太多时，会导致full gc。


<font color=red>
永久带 full gc的话建议：

1. 永久到设置大点；
2. 类不用频繁的卸载加载、如有需要尽量使用缓存。

</font>

##### 5.浮动垃圾

CMS并发回收时有对象放进老年代，但是老年代空间不够导致其他GC？
