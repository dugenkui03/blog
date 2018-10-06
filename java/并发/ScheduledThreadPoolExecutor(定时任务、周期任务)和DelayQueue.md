
##### 1.`ScheduledExecutorService`介绍

`Timer`对应的是单个后台线程，`ScheduledExecutorService`可以在构造函数中指定多个**核心线程数**，并且其最大线程数默认为`Integer.MAX_VALUE`。

对于希望某段时间后**执行一次的定时任务**和某段时间后**周期执行**(周期为两次任务开始间隔时间-可能延迟，或者下次开始距离上次任务时间)，可以使用`ScheduledExecutorService`来提交执行。

注意：

1. 调用周期任务执行方法`scheduleAtFixedRate`如果任务有延迟，两个任务是不能并行执行的，例如周期设置2s,任务执行时间3s,则周期实际表现为3s,不会出现上个任务执行到第2s时下一个任务开始并行执行；
2. 核心线程池数量一般设置成要执行周期任务的数量，如1，设置多了浪费；
3. `ScheduledExecutorService`最多可设置核心线程池、线程工厂和拒绝策略三个参数，最大线程数为`Integer.MAX_VALUE`，工作队列使用`DelayedWorkQueue`,额外线程存活时间为0s，构造器源码如下
```java
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

##### 2.周期任务使用代码示例

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //如果要提交两个周期任务并且耗时特别长，则可以使用两个线程
        ScheduledExecutorService schedulePool= Executors.newScheduledThreadPool(2);

        //只执行一次:有返回值
        Callable cal=()->1;
        int res=(int)schedulePool.schedule(cal, 1, TimeUnit.SECONDS).get();

        Runnable task=()->System.out.println("dgenkui");
        //只执行一次
        schedulePool.schedule(task,5,TimeUnit.SECONDS);

        /**
         *
         * 创建并执行一个周期性操作，该操作在给定的初始延迟后首先启用，随后在给定的时间段内启用;
         * 即执行将在initialDelay之后开始，然后是initialDelay + period，然后是initialDelay + 2 * period，依此类推。
         *  1.如果任务的任何执行遇到异常，则后续执行被禁止。 否则，任务将仅通过取消或终止执行者来终止。
         *  2.如果此任务的执行时间超过其周期，则后续执行可能会延迟，但不会同时执行。
         *
         */
        schedulePool.scheduleAtFixedRate(task,0,5, TimeUnit.SECONDS);

        /**
         * 同上也是周期动作。但周期是上一次任务结束和下一次任务开始的时间间隔
         * 如果任务的任何执行遇到异常，则后续执行被禁止。 否则，任务将仅通过取消或终止执行者来终止
         */
        schedulePool.scheduleWithFixedDelay(task,2,10,TimeUnit.SECONDS);
    }
```

##### 3.`DelayedWorkQueue`延迟队列


###### `DelayedWorkQueue`基本结构
延时任务使用的是静态内部类`DelayedWorkQueue`，其任务放在数组中，**存取都需要锁**：

```java
private RunnableScheduledFuture<?>[] queue =new RunnableScheduledFuture<?>[INITIAL_CAPACITY];

//存放
public void put(Runnable e) {
    offer(e);
}
        
public boolean offer(Runnable e, long timeout, TimeUnit unit) {
    return offer(e);
}

public boolean add(Runnable e) {
    return offer(e);
}

    public boolean offer(Runnable x) {
            //存入元素为null则抛异常
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            //获取锁
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                if (i >= queue.length)
                    grow();
                size = i + 1;
                if (i == 0) {
                    queue[0] = e;
                    setIndex(e, 0);
                } else {
                    siftUp(i, e);
                }
                if (queue[0] == e) {
                    leader = null;
                    available.signal();
                }
            } finally {
                //释放锁
                lock.unlock();
            }
            return true;
        }

//取
        //不需要移除元素
        public RunnableScheduledFuture<?> peek() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return queue[0];
            } finally {
                lock.unlock();
            }
        }
```

###### 2)延时队列和定时任务线程池的交互

- 当调用`ScheduledThreadPoolExecutor`的`scheduleXXX(Runnable command,long delay,TimeUnit unit)`方法时，会向延时队列添加了实现了`RunnableScheduledFuture`接口的`ScheduledFutureTask`。源码如下：

```java
//ScheduledThreadPoolExecutor
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit) {
    ...判断command是否为null和period是否为正...
        
    //构造含有触发时间和周期信息的任务对象
    ScheduledFutureTask<Void> sft =new ScheduledFutureTask<Void>(command,null,triggerTime(initialDelay, unit),unit.toNanos(period));
    delayedExecute(sft);
    return sft;
}

private void delayedExecute(RunnableScheduledFuture<?> task) {
    super.getQueue().add(task);
}
```
**<font color=red> 线程执行某个周期任务的流程如下:</font>**
1. 获取任务：线程从延时队列`DelayWorkQueue`中获取到期的任务`ScheduledFutureTask`；
2. 执行任务：线程执行`ScheduledFutureTask`并修改其time属性(初始延时+周期)；
3. 将任务放回延时队列中；

源码如下：
```java
        public RunnableScheduledFuture<?> take() throws InterruptedException{
            //获取锁
            final ReentrantLock lock = this.lock;//private final Condition available = lock.newCondition();
            lock.lockInterruptibly();
            try {
                //轮询检查延时队列中的任务是否到期
                for (;;) {
                    //获取队头元素
                    RunnableScheduledFuture<?> first = queue[0];
                    //如果元素为空，则放弃锁等待，直到其他线程放进元素并唤醒此线程时，开始下一轮检查
                    if (first == null)
                        available.await();
                    else {
                        //注意getDelay()是返回ScheduleFutureTask的time属性和当前时间的纳秒差值
                        long delay = first.getDelay(NANOSECONDS);
                        if (delay <= 0)//time-now()<=0标示到达了开始执行任务的时间，则返回任务
                            return finishPoll(first);
                        //否则在等待一段时间
                        else {
                            available.awaitNanos(delay);
                        }
                    }
                }
            } finally {
                //如果队列中含有任务元素，则唤醒其他等待获取任务的线程，并放弃锁
                if (queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }
        

//执行任务
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
                //执行任务
                ScheduledFutureTask.super.run();
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();//设置下次执行时间
                reExecutePeriodic(outerTask);//重新返回延时队列
            }
        }
//设置下次执行时间
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                time += p;
        }
//放回延时队列
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
```

###### 3)`ScheduledFutureTask`

`ScheduledFutureTask`是延时队列中的任务元素，主要变量有：
1. `long time`：第一次执行的延时；
2. `long sequenceNumber`：任务被添加到`ScheduledThreadPoolExecutor`中的序号；
3. `long period`：周期；

队列中的任务会先比较`time`，如果相同则比较`sequenceNumber`，都是小的先执行。