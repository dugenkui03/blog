
###### single：可以指定线程工厂，核心线程和最大线程数为1
```java
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));

```

###### Fixed：可以指定固定的线程数和线程工厂
```

    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

###### cache:可以指定线程工厂，核心线程数为0，最大线程数为Integer.MAX_Value

工作队列为`SynchronousQueue`，他没有元素存储空间，而是维护了一组线程，这些线程等待将队列移除或插入，生产者直接将元素交付给消费者。因为没有数据存储功能，所以put()和take()会一直阻塞。

```
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```

###### ScheduledThreadPoolExecutor：可以指定核心线程数和线程工厂，最大线程数为Integer.MAX_VALUE

工作队列是`DelayedWorkQueue`:

`executor.schedule(task, i, TimeUnit.SECONDS);`指定任务提交时间

#####
如果核心线程数和最大线程数都是1，那么就是`newSingleThreadScheduledExecutor`