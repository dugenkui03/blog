#### **待更新**

 1. java.lang.SecurityManager;


#### **Thread中基本方法**
##### **1.void suspend()方法**
　　这是一个过时的方法，与void resume()搭配使用来暂停和唤醒一个线程。这两个方法有可能造成独占和不同步的问题—详见《java多线程编程核心技术》。方法源码如下：
```
    @Deprecated
    public final void suspend() {
        checkAccess();
        suspend0();
    }
```
　　首先当前线程的checkAccess()方法被调用，其有可能导致SecurityException。如果当前线程时活跃着的，则挂起，调用resume()方法激活。相关方法checkAccess()和suspend0()源码如下：
```
/**
 *检测当前运行的线程是否有权限修改这个线程
 *    比如main线程使用代码someThread.suspend()来修改someThread线程；
 *如果有 安全管理器/SecurityManagy，则将被修改的线程作为参数调用安全管理器的    
 *checkAccess方法。
 *
 *如果当前线程没有权限修改被修改的线程，则抛出SecurityException异常。
 */
    public final void checkAccess() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkAccess(this);
        }
    }
```
　　native指Java调用本地方法，通常为非java语言实现
```
    private native void suspend0();//原来源码名称后缀数字的是这个意思
```
　　resume()方法同理，也是一个过时的唤醒线程的方法。如果当前线程没有权限修改被唤醒的线程，则抛出SecurityException异常
　　
```
    @Deprecated
    public final void resume() {
        checkAccess();
        resume0();
    }
```
##### **2.`void yield()`**
`Thread.yield()`是指当前线程已经完成重要操作，建议线程调度器将cpu资源从转移给另一个线程：
　　
```
public static native void yield();
```
##### **3.线程优先级**
　　线程优先级为1~10，数字越高代表级别越高优先级越高。Thread类中设置了三个默认的优先级属性：
　　
```
    /**
     * The minimum priority that a thread can have.
     */
    public final static int MIN_PRIORITY = 1;

   /**
     * The default priority that is assigned to a thread.
     */
    public final static int NORM_PRIORITY = 5;

    /**
     * The maximum priority that a thread can have.
     */
    public final static int MAX_PRIORITY = 10;
```
当我们在使用void setPriority(int newPriority) 设置线程的优先级时，数字必须在这个区间。方法源码如下：
```
/**
 *final方法：不能被改写；final类不能被继承；
 *
 *
 *
 */
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();//检查当前线程是否有修改此线程优先级的权限；
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {//优先级必须在[1,10]之间，否则抛异常：非法参数
            throw new IllegalArgumentException();
        }
        //fixme 返回被修改线程所在的线程组，如果被修改线程结束则返回null。如果要设置的值大于所在组线程的可以有的最大值，则自动降低为当前线程组已有的最大值。
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);
        }
    }
```
　　这里有一个需要注意的点：<font color=red>**如果要设置的值大于所在组线程的可以有的最大值，则自动降低为当前线程组已有的最大值。**</font>除此之外：
　　1.线程具有**继承性**，即被创建的线程与其创建者具有相同的优先级；
　　2.线程优先级即具有“规则性”，又具有“随机性”，即优先级高的线程有限执行，但其run()方法中任务又不一定是最先执行完。cup只是尽量将资源让给优先级比较高的线程，但是代码先运行、优先级高的线程不一定最先执行完。
　　综上，线程的优先级具有<font color=red>**1.组自贬性；2.继承性；3.随机性**</font>

##### **4.守护进程：setDaemon(boolean )**
　　守护进程也可理解为“保姆进程”，**当所需要服务的“雇主进程们”结束了，保姆进程自动销毁**。最典型的守护进程是GC垃圾回收器。调用Thread 的setDaemon(boolean ）方法传参true将进程设置为守护进程，此方法只能在start()调用之间调用，即进程为**非存活**的状态下。方法源码如下：
```
    public final void setDaemon(boolean on) {
        checkAccess();
        if (isAlive()) {//如果当前线程已经通过start()方法激活，则不能在更改其类型。
            throw new IllegalThreadStateException();
        }
        daemon = on;//daemon是Thread类变量，标识其是否是守护进程
    }
```
[补充]:IDEA中变量上右键“Analyze”的子菜单下“Analyze Data Flow to/from Here”可以查看变量的引用情况。

##### 五.停止线程:做标记，三方法抛异常清除，短方法清除

停止一个线程使用`interrupt()`方法
除非线程要**中断本身**。此操作是**为当前线程做标识“可以停止”**，而非真正的停止线程。

如果线程在调用`wait()、join()、sleep()`及其各种变参多态函数后，再调用`interrupt()`方法会清空其`interurpted`的状态，并抛出`InterruptedException`异常。

线程是否是“可停止状态”可以调用两个方法：
1. `boolean interrupted()`:会清除“可停止状态”，测试的是当前线程；

2.`boolean isInterrupted()`：不会清除状态，测试的是Thread对象；


```
//参数表示返回其当前状态后是否将其重新设置为“不可停止”
private native boolean isInterrupted(boolean ClearInterrupted);
```
两个方法源码如下：

```
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }

    public boolean isInterrupted() {
        return isInterrupted(false);
    }
```
　　停止线程[书中](https://item.jd.com/11701869.html)讲解了三种方式：

 1. 异常法；
 2. 在沉睡中停止；
 3. 在这里不做讲解的stop()暴力停止；


异常法即当我们使用interrupted()（当前current线程）已经标记为“可停止”时，可以在run()方法中抛出异常并在run()方法的最后捕获异常，以此来停止线程。示例代码如下：
```
public DemoClass extends Thread{
	@Override
	public void run(){
		try{
			//doSomeThing
			if(this.interrupted()){
				throw new InterruptedException();
			}
		}catch(InterruptedException e){
			//doSomeThing
		}
	}
}
```
　　正如我们上边讲的，如果线程调用wait()、join()、sleep()后再调用interrupt()方法，会清空停止状态并抛异常InterreptedException。使用这个方法也可以停止线程，其实与第一个差不多，我觉得。
##### <font color=red>**六.获取线程id**</font>：long getId();
##### <font color=red>**七.让线程休眠指定时间**</font>
　　调用sleep(long million)或者sleep(long millis,int nans)可以使调用方法的线程休眠指定时间，方法可能抛出InterruptedException和IllegalArgumentException，源码如下：
```
//使当前线程休眠指定毫秒：1秒=1000毫秒
    public static native void sleep(long millis) throws InterruptedException;

/**
 *使线程休眠指定毫秒+纳秒，其实精确不了纳秒的级别：
 *  1.如果纳秒参数>500,000则休眠时间增加1毫秒（相当于向上取整）;
 *  2.如果毫秒参数为0而且纳秒参数不为0，则休眠1毫秒；
 *  3.纳秒级别的参数不能超过1毫秒,即999，999；
 *  1ms=1000,000ns（纳秒）
 *
 **/
    public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {//纳秒级别的参数不能超过1毫秒
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        sleep(millis);
    }
```
##### <font color=red>**八.检测线程是否“存活”**</font>
　　方法源码如下，如果当前线程**正在运行或者准备运行都会返回true**：
```
 public final native boolean isAlive();
```
##### <font color=red>**九.返回当前正执行线程的引用**</font>
```
    public static native Thread currentThread();
```

　　

 　
　　

　　
　　

