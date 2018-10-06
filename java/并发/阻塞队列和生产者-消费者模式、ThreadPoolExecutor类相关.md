#### 一.引言

##### 1.1 优势：解耦和阻塞队列方便模式的实现

生产者-消费者模式将**找出完成的工作**和**执行工作**两个过程分离，消除了**生产者和消费者之间的代码依赖性**，并且将**生产数据与使用数据的过程解耦，简化工作负载的管理，这两个过程处理数据的速度可能不同**。

阻塞队列的阻塞方法put()、take()和定时的offer()、poll()方法以及一些放不进可以立即返回布尔值的方法可以方便生产者-消费者模式的实现。

##### 1.2 java中的生产者消费者模式

java源码中最典型的生产者-消费者模式就是线程池和工作队列的组合。在第二节详细展开。

##### 1.3 java中的阻塞队列

java中的阻塞队列有五个：`LinkedBlockingQueue、ArrayblockingQueue、PriorityBlockingQueue和SynchronizedQueue以及延迟队列DelayedWorkQueue`:
1. `LinkedBlockingQueue、ArrayblockingQueue、PriorityBlockingQueue`可以用于单线程线程池和固定线程现场池的工作队列实现；
2. `SynchronizedQueue`可以用于缓存线程池的工作队列的实现，此队列不保存任务元素，而是维护一组线程——生产者线程和消费者线程需要**直接交付**，此队列适用于有足够多的消费者并且总是至少有一个生产者准备交付时的情况。
3. `DelayedWorkQueue`：定时任务线程池使用的是此队列；
4. 此外还有1.8中的`work-stealing`线程池，使用的是双端队列；


#### 二.阻塞队列和生产者-消费者模式

生产者通过`void execute(Runnable command) `将任务提交到任务队列，生产者从任务队列中取得任务进行相应处理。这里以`ThreadPoolExecutor`为例，解释阻塞队列、生产者-消费者的实现。

##### 2.1 相关队列

`workQueue`是生产者消费者入队出队数据的队列。

```java

    /**
     * fixme 用于保存任务并将任务交给worker线程的队列。
     * 不要求workQueue.poll()一定返回null来表示队列为空，
     */
    private final BlockingQueue<Runnable> workQueue;
    
    /**
     * Set containing all worker threads in pool. Accessed only when
     * holding mainLock.
     * 包含所有工作者线程的集合。fixme 只有持有mainLock锁的时候才能访问
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();
```

##### 2.2 提交任务+元素入队

线程池`ThreadPoolExecutor`使用`execute(Runnable)`提交任务，步骤有三：
1. 如果当期那活跃线程数小于核心线程数，则新启线程并执行给定的任务；
2. **如果线程数量大于核心线程数，则将任务入队**；
3. 失败的话启动非核心线程，并执行参数任务。

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        //获取活跃的线程数量
        int c = ctl.get();
        //fixme 步骤一：如果活跃的线程数量小于核心线程数量，启动一个消费者线程执行任务：核心线程数量在ctl的低27位存放
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))//如果为真，使用核心线程数作为边界，FALSE使用最大线程作为边界
                return;
            c = ctl.get();
        }
        //RUNNING<SHUTDOWN<STOP<TIDYING<TERMINATED。fixme 步骤二：如果线程池处于运行态而且成功将制定元素插入到workQueue
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //如果当前线程不是RUNNING状态而且从队列中成功移除参数任务，调用RejectedExecutionHandler处理此任务：void rejectedExecution(Runnable r, ThreadPoolExecutor executor)
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)//如果当前线程池线程数量为0，则在工作者队列workers中添加工作者线程
                addWorker(null, false);
        }
        //fixme 步骤三：启动非核心线程执行任务：addworker包括启动线程和使用线程执行给定的任务
        else if (!addWorker(command, false))
            reject(command);//失败的话交由RejectedExecutionHandler处理此不能入队的任务。
    }

```

##### 2.3 出队

出队方法`getTask()`被`runWorker()`调用，而`runWorker()`被`Worker implements Runnable`类的run方法调用，而`Worker`类对象作为其Thread类变量对象的任务参数，在`addWorker(Runnable,boolean core)`中向workers集合中添加工作者时执行。

```java
    private Runnable getTask() {
        //标识最后的出队操作是否已经超时，for循环进行到第二次是为TRUE
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            //获取有效线程数，即工作者数量
            int c = ctl.get();
            //获取运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            //仅在必要时检查队列是否为空
            //如果运行状态大于等于0（SHUTDOWN、STOP、TIDYING和TERMINATED）而且 运行状态 是TIDYING/TERMINATED 或 队列为空时
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                //减小ctl的workerCount域。这种情况只是在线程突然中断时调用
                //而其他减小ctl的workerCount域的情况是调用getTask()方法时
                decrementWorkerCount();
                return null;
            }

            //取int类型c的二进制形式低29位的数字，表示工作者(活跃线程)数量
            int wc = workerCountOf(c);

            // Are workers subject to culling(淘汰)?
            //工作者是否应该被淘汰？如果允许核心线程超时或者线程数量大于核心线程数量
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //如果活跃线程数量大于参数最大线程数量或者
            //工作者应该被淘汰而且最后一次poll()超时而且 活跃线程大于1 或者工作队列为null
            if ((wc > maximumPoolSize
                    || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
                //如果自减成功，结束方法。否则继续for循环
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                //如果允许核心线程超时或者活跃线程数量大于核心线程数量，则使用指定的超时时间从任务队列获取任务，否则阻塞获取
                Runnable r = timed ?
                        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                        workQueue.take();
                //如果成功获取任务，则返回任务
                if (r != null)
                    return r;
                //否则超时，继续for循环
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }// end of for
    }
```

[补充]:`addWorker(Runnable,boolean core)`
```java
-
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry://外循环别名
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);//保留c的高三位?，-1/0/1/2/3分别代表其状态：RUNNING<SHUTDOWN<STOP<TIDYING<TERMINATED

            // Check if queue empty only if necessary.
            //如果状态为SHUTDOWN而且参数任务为null而且任务队列不为空，则直接返回false
            if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);//工作线程数量
                //如果当前线程数量大于CAPACITY五百万的线程数量，或者大于核心线程数量/指定最大线程数量，则返回FALSE
                if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //CAS操作增加ctl的workerCount域，成功则退出外循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  //重新读取ctl域
                if (runStateOf(c) != rs)//如果运行状态改变，则重新开始外循环
                    continue retry;
                // 如果CAS因为workerCount的改变而失败了，则重新计入内循环 else CAS failed due to workerCount change; retry inner loop
            }//end of inner
        }//end of outter

        boolean workerStarted = false;
        boolean workerAdded = false;//是否成功将worker对象添加进worker集合
        Worker w = null;
        try {
            w = new Worker(firstTask);//以参数任务作为构造参数初始化Worker对象。fixme 注意worker线程变量的Runnable构造参数是此参数firstTask
            final Thread t = w.thread;
            if (t != null) {//如果worker对应的thread线程不为null
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    //检查当前线程池状态
                    int rs = runStateOf(ctl.get());

                    //如果线程池状态为运行态 或者 为SHUTDOWN状态而且给定参数为null
                    if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                        //如果worker对应线程是活跃的—调用start()，则抛异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        //持有锁的时候才能够访问workers集合：fixme 向工作者集合中添加Worker线程
                        workers.add(w);
                        //当前线程池工作者数量
                        int s = workers.size();
                        //如果当前线程池中线程数量大于之前记录，则更新
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();//fixme 开始使用新启的线程执行添加的参数任务
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)//如果任务没有成功开始
                addWorkerFailed(w);
        }
        return workerStarted;
    }

```

#### 三.`ThreadPoolExecutor`类相关

[参见github相关代码注解](https://github.com/dugenkui03/learn-java)

