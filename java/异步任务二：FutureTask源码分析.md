- 参考文献
[Unsafe方法详解](https://www.cnblogs.com/dennyzhangdd/p/7230012.html)

##### 1.`FutureTask`简介

`FutureTask`实现的`RunnableFuture`接口继承了`Runnable, Future`接口。

`FutureTask`是可取消的异步计算，是`Future`的基本实现，提供了开始、取消任务`cancel(mayInterrupted)`以及查询其计算结果的方法——`get()`方法一直阻塞获取结果(接受中断信号)，`get(time)`超时则会抛异常。

除非任务使用`runAndReset`，否则一旦计算完成，任务就不能重置或者取消了。

`FutureTask`可以使用`Callable`或者`(Runnable+result)`作为构造参数——后者也会转换成`Callable`，因为他实现了`Runnable`接口，因此可以提交给线程池执行——会执行构造参数中的任务。

##### 2.状态变迁

异步任务共有7中状态，分别是：`new、completing、normal、cancelled、exceptional、interrupting、interrupted`。

可能的状态变迁包括以下四种情况：
1. 正常完成:`new->completing->normal`;
2. 异常：`new->completing->exceptinal`;
3. 取消:`new->cancelled`;
4. 中断:`new->interrupting->interrupted`。

`run()`方法执行过程中`set`方法执行完成—正常完成则NORMAL，抛异常则`EXCEPTIONAL`;
`run()`方法中`set`执行之前如果调用`cancel(boolean)`方法并修改了任务状态，则最终状态根据参数变成`CANCELLED`或者`INTERRUPTED`。详见下节代码实现：


###### [注] 等待线程状态变迁
<font color=red>
线程可能被挂起也可能被释放(停止挂起):

- 调用`get(timed)`方法获取任务结果时，线程被挂起；
- 调用`cancel()`或者`run()`方法执行结束之后调用`finishCompletion()`时，所有等待结果的线程被释放。
</font>
##### 3.主要方法的实现

`FutureTask`主要有三个特点:1.只执行一次；2.取消；3.异步获取结果。

- `run()`方法

1. 为了保证任务只执行一次，设置准入条件`只有new状态并且成功将runner变量设置为当前线程`才可执行任务；
2. 可能的状态改变，通过准入条件后：
    - 如果正常结束，则通过setXX方法设置返回值-可能为异常，最终状态为COMPLETED或EXCEPTIONAL；
    - 如果在setXX执行之前某线程执行了`cancel(boolean)`，则setXX方法内容不能成功执行，并且`handlePossibleCancellationInterrupt`中负责将状态修改为INTERRUPTED。

**<font color=red>综上：如果正常结束则`setXX`中将状态修改为`NORMAL`或者`INTERRUPTED`;如果在`setXX`方法修改状态之前调用`cancel`方法修改了状态，则最终状态根据`cancel`参数相应的变成`CANCELLED`或者`INTERRUPTED`。</font>**
```java
    public void run() {
        /**
         * FutureTask保证任务只执行一次，因为：
         *      1. 如果任务不是新建状态(已经开始执行)
         *  或者2. 是新建状态但是设置执行callable的工作线程为当前线程失败，则返回
         *          ——fixme:在多个线程获取任务并执行时，可能两个任务并发执行到这里，则此代码保证只有一个线程成功进行下去
         */
        if (state != NEW || !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
            return;
        //以上，state是NEW并且成功将runner设置为当前线程才能继续执行

        try {
            Callable<V> c = callable;
            //state可能在其他地方被修改，比如取消、中断
            //setXX方法改变状态：completing->completed 或者 interrupting -> interrupted
            if (c != null && state == NEW) {
                V result;
                boolean ran;

                //fixme 如果任务执行过程中抛异常，则调用setException(ex)方法，否则调用set(res)方法设置结果
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            //在确定状态之前，runnable必须为null，防止并发调用run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts

            int s = state;
            if (s >= INTERRUPTING)//如果状态是INTERRUPTING和INTERRUPTED。如果是CANCELLED则不用管
                //处理可能的取消中断
                // fixme run方法中在set结果之前并没有修改状态，如果此时被cancel，则set方法不能成功执行，此处需要处理cancel方法的影响：等待程序执行到INTERRUPTED处
                //详见cancel(boolean)源码
                handlePossibleCancellationInterrupt(s);
        }
    }  
```

- `cancel(boolean mayInterruptIfRunning)`方法：执行此方法最终状态可能是`CANCELLED`或者`INTERRUPTED`(如果当前状态是NEW且此通过`CAS`成功的更新了状态，则继续执行返回true)
    1. 如果是不可中断的取消，则状态置为`CANCELLED`；
    2. 如果是可中断的取消，则状态置为`INTERRUPTING`,并在执行`t.interrupt()`后，将状态更新为`INTERRUPTED`。
     
```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        /**
         * 如果当前状态是新建、而且使用CAS操作改变了新建状态(因为其他线程也可能在修改):
         *          如果可中断则修改为中断状态，否则修改为CANCELLED状态
         */
        if (!(state == NEW &&
                UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                        mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;

        /**
         * 如果可以中断，则先中断线程，在修改当前任务对象状态值。最后 移除并signal所有等待的线程，调用done()，置空callable
         */
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    //执行具体业务逻辑（构造参数值）所在的线程。fixme：通过执行线程对象的interrupt()方法中断线程
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    //避免从排序的更新值
                    //putOrderedObject，Unsafe. putOrderedInt，Unsafe. putOrderedLong这三个方法，JDK会在执行这三个方法时插入StoreStore内存屏障，避免发生写操作重排序。
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }

```

- `get()`方法

```java
    /**
     * 阻塞一定的时间获取结果，超时则抛 TimeoutException
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
                (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    /**
     * fixme: 被get方法调用，因此方法中的线程都是等待获取结果的线程
     *
     * 等待结束或者中断\超时时抛异常
     */
    private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
        //如果调用函数get设置了等待时间，则获取需要目标时间，否则目标时间是0L
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            //get()获取结果的线程被中断_get()方法调用方法
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;

            //如果当前状态不是新建和开始状态,返回当前状态
            if (s > COMPLETING) {//NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTING、INTERRUPTED
                if (q != null)
                    q.thread = null;
                return s;
            }
            //如果当前状态是正完成COMPLETING，则让出cpu资源给同等甚至更高优先级的线程
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //fixme: 如果状态是NEW而且q为null，则将当前线程包装复制给q
            else if (q == null)
                q = new WaitNode();//fixme 线程节点等于当前线程
            // 如果任务状态为NEW而且q不为null，则：1.将waiters节点赋值给q.next，然后跟waiters节点比较是否相同(todo 肯定不同啊)，相同则将waiters复制给q
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this,waitersOffset,q.next = waiters,q);
            //如果get方法设置了超时时间；//如果已经超时，则移除当前等待线程;否则关起当前线程；
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else//如果没有设置超时时间，则直接挂起当前线程
                LockSupport.park(this);
        }
    }
```

##### 4. [`Unsafe`的基本操作](http://note.youdao.com/noteshare?id=f42ec0735efe561c644a77712b04e98f)

`FutureTask`和基本类型对应的原子类，比如`AtomicInteger`等都是通过`sum.misc.Unsafe`进行CAS操作，使用模式相同：

1. 通过`sun.misc.Unsafe.getUnsafe()`方法获取`Unsafe`对象；
2. 获取类中指定字段的与域对象,然后用其作为参数获取此**字段在Java对象中相对对象起始位置的相对偏移量**；
3. 使用`Unsafe`对象方法执行CAS或者防止指令重拍的一系列操作。示例如下：

```java

public class UnsafeDemo {

    private int status;

    private static final Unsafe UNSAFE;

    private static final long statusOffset;


    static {
        try {
            UNSAFE = Unsafe.getUnsafe();

            Class<?> clz = UnsafeDemo.class;
            statusOffset = UNSAFE.objectFieldOffset(clz.getField("status"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }


    /**
     * 使用CAS累加，并返回旧值
     */
    public int casFunc() {
        int old;
        {
            old = UNSAFE.getIntVolatile(this, statusOffset);
        }
        while (!UNSAFE.compareAndSwapInt(this, statusOffset, old, old + 1)) ;

        return old;
    }

    /**
     * getAndAddInt使用comparaAndSwapInt实现，三个参数含义分别是：
     * 要改变的对象、对象偏移量、增加的值。操作是线程安全的
     */
    public int addFunc() {
        return UNSAFE.getAndAddInt(this, statusOffset, 1);
    }
}
```

##### 源码注释
```java
package com.sankuai.meituan.waimai.algorithm.thrift;

/**
 * <p>可取消的异步计算</p>
 * A cancellable asynchronous computation.
 * <p>这个类是Future的基本实现，提供了开始、取消、查询其计算结果的方法。</p>
 * This class provides a base
 * implementation of {@link Future}, with methods to start and cancel
 * a computation, query to see if the computation is complete, and
 * retrieve the result of the computation.
 * <p>结果只有在执行结束后才能够获取：get方法将一直阻塞知道计算完成，get(time)则会超时后抛异常</p>
 * The result can only be
 * retrieved when the computation has completed; the {@code get}
 * methods will block if the computation has not yet completed.
 * <p>一旦计算完成，任务就不能重制或者取消了，除非任务使用runAndReset方法调用</p>
 * Once
 * the computation has completed, the computation cannot be restarted
 * or cancelled (unless the computation is invoked using
 * {@link #runAndReset}).
 *
 * <p>FutureTask可以用来包装Callable或者Runnable，因为他实现了Runnable接口，也可以提交给线程池执行</p>
 * <p>A {@code FutureTask} can be used to wrap a {@link Callable} or
 * {@link Runnable} object.  Because {@code FutureTask} implements
 * {@code Runnable}, a {@code FutureTask} can be submitted to an
 * {@link Executor} for execution.
 *
 *
 * <p>In addition to serving as a standalone class, this class provides
 * {@code protected} functionality that may be useful when creating
 * customized(自定义) task classes.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <V> The result type returned by this FutureTask's {@code get} methods
 */
public class FutureTask<V> implements RunnableFuture<V> {
    /*
     * Revision notes: This differs from previous versions of this
     * class that relied on AbstractQueuedSynchronizer, mainly to
     * avoid surprising users about retaining interrupt status during
     * cancellation races. Sync control in the current design relies
     * on a "state" field updated via CAS to track completion, along
     * with a simple Treiber stack to hold waiting threads.
     *
     * Style note: As usual, we bypass overhead of using
     * AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
     */

    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion(完成), state may take on
     * transient短暂的 values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate(中间状态) to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions(转变):
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */


    /**
     * 以下变量记录此类对象任务的运行状态，初始化为NEW 0：
     * 1. 执行以下方法时状态转变到最终态：set-COMPLETING,setException-EXCEPTIONAL和cancle-CANCELLED;
     * 2. 在计算过程中，状态可能短暂的变成COMPLETING()、INTERRUPTING();
     * 3. 状态转变到最终状态时使用的是低消耗的 ordered/lazy写,因为最终态不会在被写；
     * 4. 可能的转变：
     *      1）新建-》完成中-》正常结束 NORMAL；
     *      2）新建-》完成中-》异常；
     *      3）新建-》取消；
     *      3）新建-》中断中-》已中断。
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;//正常结束
    private static final int EXCEPTIONAL  = 3;//异常
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    //具体执行的任务，运行之后会置空
    private Callable<V> callable;

    /** The thread running the callable; CASed during run() */
    /**运行构造参数中callable的线程：具体业务逻辑所在。在run()方法中通过CAS操作写入**/
    private volatile Thread runner;

    /** The result to return or exception to throw from get() */
    /**get()方法返回的结果或者抛出的异常**/
    private Object outcome; // non-volatile, protected by state reads/writes
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;


    /**
     * 使用Unsafe操作 status、runner和waiters（private volatile WaitNode waiters）；
     */
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            //直接使用 Class 效果基本一致，但是这样写更加规范，在某些类型转换时可以避免不必要的 unchecked 错误
            Class<?> k = FutureTask.class;

            /**
             * 1. getDeclaredField(filedName)：获取类中指定字段的域对象；
             * 2. objectFieldOffset()方法用于获取某个字段相对Java对象的“起始地址”的偏移量。
             */
            stateOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("state"));

            runnerOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("runner"));

            waitersOffset = UNSAFE.objectFieldOffset(k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    /**
     * Simple linked list nodes to record waiting threads in a Treiber（齿轮）stack.
     * <p>链表节点是 "等待线程"</p>
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Callable}.
     *
     * @param  callable the callable task
     * @throws NullPointerException if the callable is null
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
        //Executors.callable(Runnable,Object)：构造一个执行runnable任务并且返回值是指定值的任务；
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

    public void run() {
        /**
         * FutureTask保证任务只执行一次，因为：
         *      1. 如果任务不是新建状态(已经开始执行)
         *  或者2. 是新建状态但是设置执行callable的工作线程为当前线程失败，则返回
         *          ——fixme:在多个线程获取任务并执行时，可能两个任务并发执行到这里，则此代码保证只有一个线程成功进行下去
         */
        if (state != NEW || !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
            return;
        //以上，state是NEW并且成功将runner设置为当前线程才能继续执行

        try {
            Callable<V> c = callable;
            //state可能在其他地方被修改，比如取消、中断
            //setXX方法改变状态：completing->completed 或者 interrupting -> interrupted
            if (c != null && state == NEW) {
                V result;
                boolean ran;

                //fixme 如果任务执行过程中抛异常，则调用setException(ex)方法，否则调用set(res)方法设置结果
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            //在确定状态之前，runnable必须为null，防止并发调用run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts

            int s = state;
            if (s >= INTERRUPTING)//如果状态是INTERRUPTING和INTERRUPTED。如果是CANCELLED则不用管
                //处理可能的取消中断
                // fixme run方法中在set结果之前并没有修改状态，如果此时被cancel，则set方法不能成功执行，此处需要处理cancel方法的影响：等待程序执行到INTERRUPTED处
                //详见cancel(boolean)源码
                handlePossibleCancellationInterrupt(s);
        }
    }

    /**
     * Executes the computation without setting its result, and then
     * resets this future to initial state, failing to do so if the
     * computation encounters an exception or is cancelled.  This is
     * designed for use with tasks that intrinsically execute more
     * than once.
     *
     * <p>在ScheduledThreadPoolExecutor中被调用</p>
     * @return {@code true} if successfully run and reset
     */
    protected boolean runAndReset() {
        if (state != NEW ||
                !UNSAFE.compareAndSwapObject(this, runnerOffset,
                        null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }

    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * <p>更新状态和结果变量，然后进行完成计算处理</p>
     * @param v the value
     */
    protected void set(V v) {
        /**
         * 四个参数分别是：被修改的对象、要更新的属性在内存中的位置、期待的值、要更新的值(如果是期待的值、则更新为要更新的值)
         * 更新成功则返回true，失败则返回false；
         */
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

    /**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     *  <p>更新状态和结果变量，然后进行完成计算处理</p>
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }


    /**
     * 执行此方法最终状态可能是CANCELLED或者INTERRUPTED(如果当前状态是NEW且此通过CAS成功的更新了状态，则继续执行返回true)
     *      1.如果是不可中断的取消，则状态置为CANCELLED；
     *      2.如果是可中断的取消，则状态置为INTERRUPTING,并在执行t.interrupt()后，将状态更新为INTERRUPTED。
     */
    public boolean cancel(boolean mayInterruptIfRunning) {
        /**
         * 如果当前状态是新建、而且使用CAS操作改变了新建状态(因为其他线程也可能在修改):
         *          如果可中断则修改为中断状态，否则修改为CANCELLED状态
         */
        if (!(state == NEW &&
                UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                        mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;

        /**
         * 如果可以中断，则先中断线程，在修改当前任务对象状态值。最后 移除并signal所有等待的线程，调用done()，置空callable
         */
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    //执行具体业务逻辑（构造参数值）所在的线程。fixme：通过执行线程对象的interrupt()方法中断线程
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    //避免从排序的更新值
                    //putOrderedObject，Unsafe. putOrderedInt，Unsafe. putOrderedLong这三个方法，JDK会在执行这三个方法时插入StoreStore内存屏障，避免发生写操作重排序。
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }

    /**
     * 阻塞获取结果
     *
     * 如果状态为1，即没有以任何形式结束或发起中断流程，则将当前线程放入 等待链表；
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    /**
     * 阻塞一定的时间获取结果，超时则抛 TimeoutException
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
                (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    /**
     * 任务借宿后返回结果或者抛异常；
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }

    /**
     * Protected method invoked when this task transitions to state
     * {@code isDone} (whether normally or via cancellation). The
     * default implementation does nothing.  Subclasses may override
     * this method to invoke completion callbacks or perform
     * bookkeeping. Note that you can query status inside the
     * implementation of this method to determine whether this task
     * has been cancelled.
     */
    protected void done() { }

    /**
     * CANCELLED、INTERRUPTING和INTERRUPTED都将会返回true
     */
    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    /**
     * 只要不是NEW都返回true，包括COMPLETING、NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTING、INTERRUPTED
     */
    public boolean isDone() {
        return state != NEW;
    }

    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                //Thread.yield()是指当前线程已经完成重要操作，建议线程调度器将cpu资源从转移给同等或更高优先级的处于就需状态的线程
                //yield()不会阻塞该线程，它只是将该线程从运行状态转入就绪状态。
                Thread.yield(); // wait out pending interrupt
                //sleep()方法会将线程转入time waiting状态，直到阻塞时间结束，才会转入就绪状态

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }



    /**
     * Removes and signals all waiting threads, invokes done(), and nulls out callable.
     * <p>移除和signal所有等待的线程，调用done()，置空callable</p>
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            //移除所有等待的线程，如果移除成功， todo 线程等待什么
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc fixme：引用存活的对象的对象也不会进行垃圾回收
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }

    /**
     * fixme: 被get方法调用，因此方法中的线程都是等待获取结果的线程
     *
     * 等待结束或者中断\超时时抛异常
     */
    private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
        //如果调用函数get设置了等待时间，则获取需要目标时间，否则目标时间是0L
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            //get()获取结果的线程被中断_get()方法调用方法
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;

            //如果当前状态不是新建和开始状态,返回当前状态
            if (s > COMPLETING) {//NORMAL、EXCEPTIONAL、CANCELLED、INTERRUPTING、INTERRUPTED
                if (q != null)
                    q.thread = null;
                return s;
            }
            //如果当前状态是正完成COMPLETING，则让出cpu资源给同等甚至更高优先级的线程
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //fixme: 如果状态是NEW而且q为null，则将当前线程包装复制给q
            else if (q == null)
                q = new WaitNode();//fixme 线程节点等于当前线程
            // 如果任务状态为NEW而且q不为null，则：1.将waiters节点赋值给q.next，然后跟waiters节点比较是否相同(todo 肯定不同啊)，相同则将waiters复制给q
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this,waitersOffset,q.next = waiters,q);
            //如果get方法设置了超时时间；//如果已经超时，则移除当前等待线程;否则关起当前线程；
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else//如果没有设置超时时间，则直接挂起当前线程
                LockSupport.park(this);
        }
    }

    /**
     * Tries to unlink a timed-out or interrupted wait node to avoid
     * accumulating garbage.  Internal nodes are simply unspliced
     * without CAS since it is harmless if they are traversed anyway
     * by releasers.  To avoid effects of unsplicing(拼接) from already
     * removed nodes, the list is retraversed in case of an apparent
     * race.  This is slow when there are a lot of nodes, but we don't
     * expect lists to be long enough to outweigh higher-overhead
     * schemes.
     *
     * <p>尝试 unlink 一个超时或者中断的wait node 来避免内存垃圾的累加。</p>
     */
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset, q, s))
                        continue retry;
                }
                break;
            }
        }
    }
}
```