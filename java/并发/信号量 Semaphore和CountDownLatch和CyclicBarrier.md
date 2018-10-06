
##### 0.引言

1. `Semaphore`可以理解为可以重复加锁的`ReentrentLock`,加锁释放分别是`void acquire()`、`void release()`，可以实现资源池和有界阻塞队列——其存放元素的视图必须是同步的；
2. `CountDownLatch`可以实现两类任务执行的先后顺序，其构造参数也有int计数器n，调用此对象用`await()`的线程会进入阻塞状态，等到（任意线程）在此此对象执行n此 `countDown()`才会恢复运行；
3. `CyclicBarrier(int n,new Runnable())`没有“减数方法”，而是**先执行n此`await()`后，会执行`new Runnable()`任务，然后恢复挂起的线程**。



##### 1. Semaphore

###### 简介

Semaphore中管理者一组虚拟的许可，许可的初始数量可以使用构造参数指定`Semaphore sem=new Semaphore(10)`，`void acquire()`方法阻塞获取到许可，如果许可都被占用，`void release()`方法释放许可。


semaphore可以用来实现**资源池**和**有界阻塞容器**。注意：
1. `Semaphore`与线程是不关联的；


###### 使用示例

- 资源池
```

class GoWC implements Runnable{
    private static AtomicInteger atomicInteger=new AtomicInteger(1);

    private Semaphore semaphore;

    public GoWC(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void run() {
        try {
            semaphore.acquire();
            TimeUnit.SECONDS.sleep(1);
            System.out.println(atomicInteger.getAndIncrement()+":sa la la,wondful");
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
public class SemaphoreDemo {
    private static Semaphore semaphore=new Semaphore(2);

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 10; i++) {
            new Thread(new GoWC(semaphore)).start();
        }
    }
}
```

- 有界容器：边界值就是许可数量
```
//以下实现可以保证容器元素数量不超过构造参数值
public class SemaphoreBoundContain <T>{
    private final Set<T> set;

    private final Semaphore semaphore;

    //容器可存放元素数量
    public SemaphoreBoundContain(int bound) {
        //返回hashSet的线程安全视图
        this.set=Collections.synchronizedSet(new HashSet<>());
        semaphore=new Semaphore(bound);
    }

    //在set中加入元素t，并返回是否能够加入
    public boolean add(T t) throws InterruptedException {
        semaphore.acquire();
        boolean wasAdded=false;
        try{
            wasAdded=set.add(t);
            return wasAdded;
        }finally {
            if(!wasAdded){
                //如果没有加入，则释放许可
                semaphore.release();
            }
        }
    }
    
    //移除一个元素，释放一个许可
    public boolean remove(T t){
        boolean wasRemove = set.remove(t);
        if (wasRemove) {
            semaphore.release();
        }
        return wasRemove;
    }
}
```

##### 2. CountDownLatch

###### 简介 

如果当前线程执行到某一步需要`thread_pre`执行完毕后再继续执行时，可以调用`thread_pre.join()`。但是如果“当前线程”和“thread_pre”是集合的话，而且两者内部有执行又是并行关系，显然`join()`方法行不通。此时用到`CountDownLatch`。

`CountDownLatch`类初始化构造参数是==整数x==。而在`CountDownLatch`类某个对象上调用`await()`会进入到阻塞状态，而且知道其他线程在此对象上调用 x 次方法将x置为0。一个线程可以调用多次`countDown()`方法。

###### 示例代码

```
class PreTask implements Runnable{
    CountDownLatch countDownLatch;

    public PreTask(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        System.out.println("pretask working！！\n smaller than "+countDownLatch.getCount());
        countDownLatch.countDown();
//        countDownLatch.countDown(); 每个线程是可以调用多次的
    }
}

class WaiteTask implements Runnable{
    CountDownLatch countDownLatch;

    public WaiteTask(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        System.out.println("waite for pretask");
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("waittask executing");
    }
}

public class CountDownDemo {
    public static void main(String[] args) {
        ExecutorService executorService= Executors.newCachedThreadPool();

        CountDownLatch countDownLatch=new CountDownLatch(100);//执行100次 countdown()方法才可以唤醒所有等待线程；

        for (int i = 0; i < 10; i++) {
            executorService.execute(new WaiteTask(countDownLatch));
        }

        for (int i = 0; i < 100; i++) {
            executorService.execute(new PreTask(countDownLatch));
        }

        executorService.shutdown();
    }
}
```

##### 3.CyclicBarrier

如名：循环的栅栏。其构造参数是

`CyclicBarrier cyclicBarrier=new CyclicBarrier(n,new Runnable());`
- 调用此对象的线程都将会被挂起来，直到第n次调用后所有线程被唤醒；
- 第n次调用`await()`的方法会执行`new Runnable()`任务，然后才唤醒所有挂起的线程。


- 示例代码
```
class Swordsman implements Runnable{
    CyclicBarrier cyclicBarrier;

    public Swordsman(CyclicBarrier cyclicBarrier) {
        this.cyclicBarrier = cyclicBarrier;
    }

    @Override
    public void run() {
        System.out.println("剑客"+Thread.currentThread().getId()+"到达华山");
        try {
            cyclicBarrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        ExecutorService executorService= Executors.newCachedThreadPool();

        CyclicBarrier cyclicBarrier=new CyclicBarrier(3,()->{
            Thread.sleep(10000);
            System.out.println("剑客"+Thread.currentThread().getId()+"开始决斗");
        });

        for (int i = 0; i < 3; i++) {
            executorService.execute(new Swordsman(cyclicBarrier));
        }

        executorService.shutdown();
    }
    
}

output：
剑客12到达华山
剑客11到达华山
剑客13到达华山
剑客13开始决斗

```






