
本部分内容来自于《java并发编程实战》6.2小节。觉得讲的很好但是很多地方还是不太理解，觉得还是需要扎实的功底和比较多的相关经验才能够透彻的理解。
#### **前言**
　　任务是一组逻辑工作单元，线程则是使任务异步执行的机制。<font color=red>**java类库中，线程执行的主要抽象是Executor，而不是Thread**</font>。Executor源码及：
```
package java.util.concurrent;

public interface Executor {
    void execute(Runnable command);
}
```
###### **Executor接口主要实现接口/类**
![这里写图片描述](http://img.blog.csdn.net/20171217134011692?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmVpcmR1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
由上图可知Executor接口的优点：

 1. 为异步任务执行框架提供了基础，支持多种不同类型的任务执行策略；
 2. 用Runnable表示任务——任务，即一组逻辑工作单元；<font color=red>**Executor基于生产者-消费者模式：提交任务的操作是生产者，执行任务的线程是消费者**</font>；
 3. 提供了对**生命周期的支持**、**统计信息收集**、**应用程序管理机制**和**性能监视**等机制；
##### **一.基于Executor的web服务器**

　　基于线程池的web服务器：
```
public class TaskExecutionWebServer {
    private static final int NTHREADS=100;
    private static final Executor exec
            = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket=new ServerSocket(80);
        while(true){
            final Socket connection=socket.accept();
            Runnable task=(()-> System.out.println("do somethings"));
            exec.execute(task);
        }
    }
}
```
　　改变Executor实现或配置所带来的影响要远远小于**改变任务提交方式带来的影响**
　　为每个请求启动一个新线程的Executor：
```
public class ThreadPerTaskExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        new Thread(command).start();//异步执行
    }
}
```
　　同步方式执行所有任务的Executor
```
public class WithinThreadExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        command.run();//同步执行
    }
}
```
##### **二.执行策略**
　　通过将任务的**提交**和**执行**解耦，从而可以简单的修改执行策略。执行策略有“what、where、when和how”等四个方面：

 1. 在什么线程中执行任务；
 2. 按照什么顺序执行任务：FIFO、FIFO、优先级；
 3. 有多少个任务可以并发执行；
 4. 队列有多少个任务在等待执行；
 5. 系统过载需要拒绝任务时，选择哪一个拒绝，怎样通知应用程序任务被拒绝；
　　**最佳的执行策略取决于可用的计算资源和服务质量的需求。限制并发任务的数量可以确保程序不会犹豫资源耗尽而崩溃，或者在需求资源商发生竞争而影响性能。**
##### <font color=red>**三.线程池和工作队列**
　　线程池存在着许多执行任务的工作者线程（worker thread），线程多少有线程池种类来确定（比如Fixed、single、scheduled及cache），线程池中的工作者线程由线程工厂产生。
　　工作队列，在java中即保存在所有等待执行任务的阻塞队列。
　　在线程池中执行任务更优，相比于为每个人物分配一个线程。通过重用线程可以将线程创建销毁的巨大开销分摊到其处理的多个请求上，而且后续请求到达时线程已经存在，可以提高相应。恰当数量的线程可以是处理器保持繁忙，而且又不会是过多线程增多资源而耗尽内存。
##### <font color=red>**四.线程池的基本创建方式及其优缺点**
　　以下方法都是“工厂方法”：
 1. newFixedThreadPool：每提交一个任务就创建一个线程，直到最大的“固定长度”，如果有线程因为Exception而结束，线程池会补充一个线程；
 2. newCacheThreadPool：单线程的线程池/executor，如果结束则会补充一个线程。其能确保任务按照在队列里的顺序执行(FIFO/LIFO/优先级);
 3. newScheduledThreadPool：固定长度、定时执行——支持基于相对时间的调度；
##### <font color=red>**[拓展].线程工厂和线程池**
　　已知工厂模式可以减小耦合，不如对象的执行和对象的创建可以分开。线程的创建在java类库中也用到了工厂模式，ThreadFactory接口源码如下：
```
package java.util.concurrent;


public interface ThreadFactory {

    /**
     * 创建一个包含任务的线程；
     */
    Thread newThread(Runnable r);
}
```
　　以上可知java.util.concurrent.Executors中好多工厂方法创建线程池(Executor及其实现类)中用到了线程工厂，线程工厂既可以指定，也可以使用Executors中的静态内部类DefaultThreadFactory—默认工厂，实现了线程工厂，源码如下:
```
    /**
     * 定义在Executors内的
     * The default thread factory
     */
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);//线程池序列号
        private final ThreadGroup group;//线程所在线程组
        private final AtomicInteger threadNumber = new AtomicInteger(1);//线程序列号
        private final String namePrefix;//线程名前缀
		
		//默认构造函数初始化：线程组对象、命名前缀
        DefaultThreadFactory() {
	        //TODO SecurityManager是个啥玩意儿
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();//线程组引用指向其线程组对象
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";//线程前缀赋值
        }

		//线程工厂主要方法：1.创建线程对象：组信息、任务Runnable、命名；2.设置为“非守护进程”；3.设置为中等优先级；
        public Thread newThread(Runnable r) {
	        //Runnable初始化线程对象
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            //设置为“非守护进程”
            if (t.isDaemon())
                t.setDaemon(false);
            //设置为一般优先级
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```
　　以创建“定长线程池”为例，其源码如下：
```
    public static ExecutorService newFixedThreadPool(int nThreads) {
	    //底层还是一个执行长度nThreads的“线程池执行器”
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
调用的构造函数如下：
```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);//由此可知调用了Executors中实现了ThreadFactory接口的默认工厂
    }
```

```
 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
##### **<font color=red>五.Executor声明周期：运行、关闭和已终止**
　　ExecutorService类拓展了Executor类，添加了一些用于声明周期管理的方法：
```
package java.util.concurrent;

public interface ExecutorService extends Executor {

    void shutdown();
   
    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

　　ExecutorService生命周期的三种状态释义：

 1. 运行：ExecutorService在初始化时处于**运行**状态；
 2. 关闭是一个过程：平缓的关闭则ExecutorService不再接受新任务，并等待旧任务的执行结束；粗暴的关闭则会尝试取消所有正在运行的任务；
 3. 关闭后当所有任务执行完毕后，ExecutorService到达终止状态；
　　关闭状态下，提交的人物将由接口java.util.concurrent.RejectedExecutionHandler处理。其源码如下：
```
package java.util.concurrent;

public interface RejectedExecutionHandler {

    /**
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}

```
或者是的Executor接口的execute方法抛出一个RejectedExecutionException 异常，与其有关的方法：
```
	//等待这么长时间：长度和时间单位，如果ExecutorService终止则返回TRUE，否则返回FALSE；
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
        
    //是否处于“已终止”的状态，即所有人物都结束了
	boolean isTerminated();
	//使ExecutorService处于关闭状态
    void shutdown();
```
　　支持关闭操作的Web服务器
```
class LifecycleWebService{
	private finale ExecutorService exec=...；
	
	public void start() throws IOException{
		ServerSocket socket=new ServerSocket(80);
		while(!exec.isShutdown()){//？不是处于关闭状态？
			try{
				final Socket conn=socket.accept();
				exec.execute(new Runnable(){
					public void run { handleRequest(conn);}
				});
			}catch(RejectedExecutionException e){
				if(!exec.isShutdown())
					log("task submission rejected",e);
			}
		}
	}

	public void stop(){ exec.shutdown();}

	void handleRequest(Socket connection){
		Request req=readRequest(connection);
		if(isShutdownRequest(req))
			stop();
		else
			dispatchRequest(req);
	}
}
```
##### <font color=red>**六.周期任务和延迟任务**</font>

延迟任务和周期任务可以使用Timer或者ScheduledThreadPoolExecutor（可以通过其构造函数或者Executors.newScheduledThreadPool工厂方法来创建）。
　　Timer类只能创建一个线程，而且不会捕获异常、因此抛出未检查的异常时会终止线程的执行，不会恢复线程的执行。
　　由上图可知ScheduledThreadPoolExcutor是继承了ThreadPoolExecutor类、实现了ScheduledExecutorService接口的类。
　　Executurs中构造“周期线程池”的工厂方法只有两个，分别：可以指定保存在线程池中的线程数量—即使是空闲状态的；和指定保存在线程池中线程数量与线程工程类。<font color=red>**但是线程池中最大线程数都是Integer.MAX_VALUE，这会有内存溢出的风险。**</font><font color=blue>**因此推荐直接使用ScheduledThreadPoolExecutor构造函数——貌似他的构造函数也不支持指定最大线程数量啊**</font>。阿里手册实例如下：
　　![这里写图片描述](http://img.blog.csdn.net/20171219102007709?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmVpcmR1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)