- 需要查看编译后的汇编代码确定volatile禁止重排的效果；

##### 1.引言

JAVA内存模型JMM java memory model可以理解为特定操作协议下，对内存或者告诉缓存进行读写访问的过程抽象。协议指缓存一致性协议cache coherence protocol，比如MESI modify exclusive shared invalid。

jmm屏蔽了各种硬件和==操作系统的内存访问==差异，实现让java在各种平台下都能够达到一致的内存访问效果。则可能也是“一次编译、到处运行”的原因之一。

- 处理器、高速缓存、主存以及<font color=red>**缓存一致性协议**</font>交互图

![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fq13iwg31vj30mp09jtbz.jpg)

##### 2. jvm模型

###### 2.1 主内存与工作内存

![](https://wx3.sinaimg.cn/mw690/006Xp67Kly1fq13ogscqkj30j708odhf.jpg)

- 线程之间无法直接访问对方工作线程中数据；
- 与“java内存区域”划分的虚拟机栈、java堆没有绝对对应关系，但是主内存对应java堆，工作内存对应虚拟机栈——虚拟机让工作内存放置于寄存器和cache；
- <font color=red>**save 和load等操作实现硬件层面的缓存一致性协议。**</font>初次之外还有 lock、unlock、use、assign、store、write方法。


###### 2.2 内存4种交互即其6种规则

**内存交互有以下4大类**：

1. lock、unlock，作用于==主内存，由线程发起==，把变量表示为一条线程独占的状态，后者是释放锁；
2. read、load：主要作用于==工作内存==，前者是从主存读取数据到工作内存，后者是将主内存得到的变量放进工作内存副本；
3. use、assign：主要有线程的==执行引擎==负责：前者执行引擎从工作内存获取一个变量，后者将执行引擎修改的值覆盖到工作内存中；
4. store、write：主要作用于==主内存==，前者将工作内存变量送到主内存，后者将此变量放到主内存变量中；

**内存交互时要遵守以下6项规则**：
1. 不允许只搬运不接受，即通过read或store将数据放入到工作内存或主内存，但是却没有通过load和write写入内存变量；
2. assign（可能发生多次）后的数据必须放回主存，而没有经过执行引擎处理过的数据则没有必要通过store、write写回主存；
3. 对变量执行lock时，会清空所有工作内存中此变量的值——因此执行引擎要用的话必须重新加载，并且只会有一个线程加载成功；
4. 对变量执行unlock时，必须通过执行store、write将变量同步回主内存；
5. 同一时刻变量只能被一个线程执行lock操作，并且此线程可以执行多次lock（对应MESI的exclusive状态），但是==执行相同次数的unlock后变量才会被解锁==；
6. 变量只能诞生于主内存；

##### 3 volatile变量的特殊规则

volatile变量主要有两个特性：**可见性和禁止指令重排优化**
1. 对所有线程都是<font color=red>**可见的**</font>，即值一经修改即可告知所有线程。**但volatile不是线程安全的**，比如自增操作两个线程读取到旧值，然后执行自增后写入到原变量；

volatile可见性**控制并发**的场景如下：**停止所有线程中执行的doWork()方法**；
```
volatile boolean shutdownFlag;

//任意线程执行此方法，doWork便停止调用
public void shutdown(){
    shutdownFlag=true;
}

public void doWork(){
    while(!shutdownFlag){
        //doSomething
    }
}

```
2. volatile禁止指令重排

一下是指令重排序引发的错误：
```
//线程共享变量
boolean ready=false;

//thread_a执行代码：两句代码相互没有任何依赖
"做了一些thread_b用到的前期准备工作"
ready=true;

//thread_b

while(!read){ sleep(); }
"read准备成功表示为TRUE，开始工作"
```

重排后的指令可能使得`ready=true;`在`"做了一些thread_b用到的前期准备工作"`之前执行，导致thread_a的准备工作没有完成thread_b就开始执行，导致出错。**使用volatile修饰ready可禁止重排**。

- Volatile的使用场景之一是double check lock单例模式
```
public class SingleTon{
    private volatile static Singleton instance;
    
    public static Single getInstance(){
        if(instance==null){
            synchronized(Singleton.class){
                if(instance==null){
                    instance=new Singleton();
                }
            }
        }
        return instance;
    }
}
```
- 内存屏障，汇编 lock前缀代码实现：lock修饰的语句是内存屏障memory barrier，后边的汇编代码不会被重排序到他前边的代码，及时没有任何依赖；
- lock 作用是本cup的cache写入主存，并引起其他cup或内核的cache无效；
- 对volatile的写操作相对较慢，因为会加入许多内存屏障保证处理器不是重排优化；

<font color=red>**可总结针对volatile的操作有如下规则**</font>：

1. volatile变量每次使用前先刷新，即read、load、use连续执行；
2. volatile变量每次修改后立即同步回主存，即assign、load、write连续执行;
3. action_a对线程对volatile变量a执行use、assign操作，action_b对volatile变量b执行use、assign操作，则action_a与action_b的执行顺序与a、b变量在主内存-工作内存中同步操作顺序相同。**保证volatile修饰的变量不会被重排序，保证代码执行顺序与程序顺序相同**。

##### 4. jmm、并发过程中怎样处理原子性、可见性、有序性

==JMM就是围绕着在并发过程中怎样处理原子性、可见性和有序性建立的==。

###### 4.1 原子性

即有些操作需要是原子操作：
1. 基本数据类型的访问可以认为是原子操作；
2. 可以通过加锁保证指令集操作的原子性；

###### 4.2 可见性

volatile、final修饰的变量的操作具备可见性，加锁代码块的操作也具备可见性，因为：
1. volatile变量的read、load、use操作和assign、store、write必须是连续的，即修改后必须立即同步会主内存，使用时必须从主内存刷新，由此保证volatile变量的可见性；
2. 同步代码块：对于同步代码块执行unlock时，其中的变量必须同步回主存——执行store、write操作。由此保证了同步代码块的可见性；
3. final修饰的变量在构造器或代码块（每次创建对象时执行）初始化后不能在被修改，有吃保证可见性；

###### 4.3 有序性ordering

有序性是指“本线程内观察则所有操作都是有序的；如果在某线程中观察另外一个线程，则所有操作都是无序的。”

前半句**有序**指的是线程内表现为串行的语义within-Thread as-if-serial semantics；后半句指的是**指令重排**和**工作内存和主内存同步延迟现象**。

java提供了volatile和同步代码块保证了 **==线程间执行的有序性==**。volatile包含了**禁止指令重排的语义**。**同步代码块**的一个时刻只允许一条线成对其执行lock操作的规则保证了持有同一个锁的两个同步代码块只能串行执行。


综上，同步代码块可以保证原子性、可见性和有序性，因此被乱用，但是只要求一个特性是，尽量使用简单的操作，因为加锁解锁性能消耗大。


##### 5. happen-before先行发生原则

happen-before是指 ==如果action_a先于action_b,则action_a的影响能够被action_b观察到，包括对共享变量的修改、发送了消息、调用了方法等。== happen-before是判断**数据是否存在竞争**、**线程是否安全**的重要依据。

？？happen-before是为了保证有序性？？


###### <font color=red>JMM中的happen-before关系</font>
JMM中有一些自然的happen-before关系，无须同步器同步。如果两个操作不在以下原则中而且无法通过以下原则推到出来，则虚拟机可以随意对他们进行重排序，两个操作的顺序性无法保证。**八种天然的happen-before先行发生关系如下：**

1. program order rule 程序执行顺序:**一个线程内**程序按照代码执行顺序——考虑到分子、循环等，应该说是**控制流顺序**；
2. Monitor lock rule 管程锁定原则/监视器锁定原则：unlock操作**先行**发生于后边的unlock操作——注意 ==**先行**是指unlock之前的所有操作的影响对于unlock之后的操作都是可见的==；
3. volatile变量规则：对于一个volatile变量的**写**操作**先行**发生于之后对这个变量的**读**操作；
4. thread start ruler 线程启动规则：start() 方法先行发生于线程中的每一个规则；
5. ？？？ thread termination ruler 线程终止规则？？？：线程中所有操作都先行发生于此线程的终止检测，我们可以通过`Thread.join()、 Thread.isAlive()的返回值手段判断线程已经终止执行`；
6. thread interruption rule 线程中断原则: 对线程`interrupt()`方法的调用先行发生于==被中断线程的**代码检测到中断事件的发生**==，可以通过`Thread.interrupted()`方法检测到是否有中断发生；
7. finalizer ruler 对象总结规则：对象初始化完成先行于他的`finalize()`方法的开始；
8. transitivity 传递性：action_a先行于action_b，action_b先行于action_c，则action_a一定先行于action_c；




