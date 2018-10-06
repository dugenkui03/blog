##### 1.前言

显式的使用Lock实现类对象加锁是解决共享资源竞争的一个方法,synchronized也可以解决共享资源竞争的问题。原子类AtomicXXX和volatile某些情况也可以解决问题。

<font color=blue>**synchronized是最常用，如果共享资源被多个任务操作且肯能被某个任务修改，则此资源应为volatile类型。**</font>

Lock对象显式的创建、锁定和释放。其主要实现类有<font color=red>**`ReentrantLock、ReadLock、WriteLock、ReadLockView和WriteLockView`**</font>，主要方法有`void lock(); boolean tryLock(); boolean tryLock(long,TimeUnit) 和 void unlock()`等，示意图如下：
![](http://wx2.sinaimg.cn/large/006Xp67Kly1fnxugfk7y2j31080ktjv7.jpg).

##### 2.`Lock`的基本使用方式

Lock对象可以获得更细粒度的控制

```
public class AttempLocking  {
    //私有
    private ReentrantLock lock=new ReentrantLock();

    public void untimed(){
        boolean captured=lock.tryLock();//fixme 尝试获取锁
        try{
            System.out.println("tryLock(): "+captured);
        }finally {
            if(captured)//fixme 如果获取锁，这在finally中释放
                lock.unlock();
        }
    }

    public void timed(){
        boolean captured=false;
        try{
            captured=lock.tryLock(2, TimeUnit.SECONDS);//指定等待锁的时间
        }catch (InterruptedException e){
            throw new RuntimeException(e);
        }

        try{
            System.out.println("tryLock(2,TimeUnit.SECONDS): "+captured);
        }finally {
            if(captured)
                lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        final AttempLocking al=new AttempLocking();
        //fixme 相同对象执行两个方法，他们共享lock对象的
        al.untimed();
        al.timed();

        //开始一个获取锁的进程
        new Thread(){
            {setDaemon(true);}//设置为守护进程，即主进程结束，守护进程也结束

            @Override
            public void run() {
                //一般情况下获取锁以后都要在finally中unlock();
                al.lock.lock();
                System.out.println("acquired");
            }
        }.start();
        
        TimeUnit.SECONDS.sleep(1);//当前线程停止一会儿，等待上句代码启动的线程获取到al的锁
        al.untimed();
        al.timed();
    }
}

```

##### 3.`Lock`+`Condition`的使用方式
java1.6之后，内置锁synchronized和Lock性能相近，但是Lock可以实现无条件的、可轮询的、定时的和可中断的锁获取操作，所有加锁的方式都是显式的。

其中`Condition`对象可以实现线程的挂起和唤醒，他与内置锁的对应关系如下：
- `Object`中的`wait()`方法相当于`Condition`类中的`await()`方法，`wait(long,timeout)`相当于`await(long,timeunit)`方法；
- `Object`中的`notify()、notifyAll()`相当于`Condition`类中的`signalAll()`方法；

<font color=red>**使用`Condition`可以仅仅通知部分使用同一个监视器对象的挂起的线程**</font>，示例如下：
```
public class ConditionDemo {
    private Lock lock=new ReentrantLock();//对象私有
    public Condition conA=lock.newCondition();
    public Condition conB=lock.newCondition();

    public void awaitA(){
        try{
            lock.lock();//加锁
            conA.await();//fixme 用lock对应的Condition对象conA挂起，则用conA才能够唤醒此处挂起的线程
        }catch (InterruptedException e){
            e.printStackTrace();
        }finally {
            lock.unlock();//fixme 一定要记住释放锁；
        }
    }

    public void awaitB(){
        try{
            lock.lock();
            conB.await();
        }catch (InterruptedException e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public void signalAll_A(){
        try{
            lock.lock();
            conA.signalAll();// fixme 唤醒所有用lock加锁且用conA挂起的任务
        }finally {
            lock.unlock();
        }
    }
}
```

##### 4.`lock()、tryLock()和tryInterruptibly()`的区别
- `void lock()`：调用后一直阻塞知道获取锁，对应==无限制锁==；
- `boolean tryLock()`：尝试获取锁，不能则返回FALSE，对应==定时锁==，通过while可作为==轮询锁==；
- `void lockInterruptibly()`：同`lock()`，但是接受中断信号，对应==可中断锁==；
