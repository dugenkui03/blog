##### 1.引言
阿里《java编程桂法》中讲到最好手动创建线程池ThreadPoolExecutor，这样可以指定**核心线程数量、最大线程数量、非核心线程存活时间、线程工厂和工作队列以及“溢出任务”处理方式等。**

前边已经讲过线程池的好处：
1. 通过重复利用线程池中的线程来降低创建和销毁线程产生的资源消耗；
2. 线程池的线程可以立即执行任务，提高相应速度；
3. 方便同一管理：


##### 2.重点

0. ThreadPoolExecutor类的execute(new Runnable())并非是执行任务，而是将任务提交到工作队列，等线程池中的线程去执行任务；
1. ThreadPoolExecutor类的execute(new Runnable())不是采用阻塞的方式向工作队列中插入任务的，当工作队列没有空间而且线程数量达到最大值时会抛异常；
2. 自定义线程工厂可以自定义线程构造参数内容：比如名字、线程所在线程组
    1. shutdown()用于任务执行完之后关闭；
    2. shutdownNow()用于强制关闭任务；
3. 线程池调用`shutdown()`或者`shutdownNow()`方法才会关闭所有线程：前者是等待任务完成后关闭相应线程，后者则强制关闭抛异常；
```
public class TestQueue {

    public static void main(String[] args) throws InterruptedException {
        //自定义工作队列，用于添加提交的任务:可以自定义队列容量
        LinkedBlockingQueue<Runnable> workQueue=new LinkedBlockingQueue<>(6);

        //自定义的线程工厂，可以自定义线程构造参数内容：比如名字、线程所在线程组
        ThreadFactory namedThreadFactory=new ThreadFactory() {
            private final AtomicInteger threadNumber=new AtomicInteger(1);//线程号
            private final String preName="thread-";//县城名前缀
            @Override
            public Thread newThread(Runnable r) {
                Thread t=new Thread(r,preName+threadNumber.getAndIncrement());//累加
                if(t.isDaemon())
                    t.setDaemon(false);
                return t;
            }
        };

        ThreadPoolExecutor threadPoolExecutor=
                new ThreadPoolExecutor(1,1,0L, TimeUnit.MILLISECONDS,workQueue,namedThreadFactory);

        for (int i = 0; i < 6000; i++) {
            threadPoolExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    try{
                        TimeUnit.MILLISECONDS.sleep(5000);
                        System.out.println(Thread.currentThread().getName());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }

//        threadPoolExecutor.shutdownNow(); 不等任务执行完，会抛异常
        threadPoolExecutor.shutdown();

    }
}
```

##### 4.补充
正如之前所说，当用`execute(new Runnable())`提交一个任务时，线程池可能检查**核心线程数量、工作队列和最大线程数量**三个参数，步骤如下：
1. 如果当前运行的线程数量小于核心线程数量(corePoolSize),则尝试开启一个新线程并将任务作为他的第一个任务；
2. 如果核心线程数量饱和，则进入尝试将任务加入任务队列；
3. 如果任务队列已满，则查看当前线程数量是否已经超过线程池的最大线程数量(maximumPoolSize),没有则启动新线程执行任务，否则执行饱和策略(RejectedExecutionHandler);