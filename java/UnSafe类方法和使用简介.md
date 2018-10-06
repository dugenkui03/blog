- 参考文献

[Unsafe源码分析](https://www.cnblogs.com/dennyzhangdd/p/7230012.html)


##### 1. CAS方法
交换int、long和对象：
```java
    public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```
##### 2. 指令重拍
防止指令重排的写操作，JDK会在执行这三个方法时插入StoreStore内存屏障，避免发生写操作重排序：
```java
    public native void putOrderedObject(Object var1, long var2, Object var4);

    public native void putOrderedInt(Object var1, long var2, int var4);

    public native void putOrderedLong(Object var1, long var2, long var4);
```
##### 3. 获取没有访问权限的实例变量`getObject(Obj,offset)`
1. 非系统类需要通过反射获取`Unsafe`实例；
2. 通过`Unsafe的 getObject(Obj,offset)`方法获取私有变量值。示例如下：
```java
public class PrivateGetterDemo {
    private static final Unsafe UNSAFE;
    private static final long noGetNameOffset;

    static{
        try{
            //fixme 非系统类使用反射获取UNSAFE对象
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            UNSAFE = (Unsafe) f.get(null);

            noGetNameOffset=UNSAFE.objectFieldOffset(PrivateDemo.class.getDeclaredField("noGetname"));
        }catch (Throwable e){
            throw new Error(e);
        }
    }

    public static void main(String[] args) {
        //fixme 理论上没有访问权限的实例变量通过Unsafe获取
        PrivateDemo pd=new PrivateDemo("pd");
        System.out.println(UNSAFE.getObject(pd, noGetNameOffset));
    }
}
```
##### 4. 线程的挂起和重新开始：可以指定挂起时间
线程挂起和终止线程挂起方法：LockSupport类实现使用这两个方法；
```java
//线程调用该方法，线程将一直阻塞直到超时，或者是中断条件出现；
//1.isAbsolute指使用的时间是相对时间还是绝对时间，false为相对时间；
//2.时间单位是纳秒；
public native void park(boolean isAbsolute, long time);  

//终止挂起的线程，恢复正常.
public native void unpark(Object thread);  
```

##### 5.锁定对象

使用一个监视器对象进行加锁、解锁。

```java
55     //锁定对象，必须是没有被锁的
56     public native void monitorEnter(Object o);  
57       
58     //解锁对象  
59     public native void monitorExit(Object o);  
60       
61     //试图锁定对象，返回true或false是否锁定成功，如果锁定，必须用monitorExit解锁  
62     public native boolean tryMonitorEnter(Object o);  
```

##### 6. 创建实例对象

```java
82     //创建一个类的实例，不需要调用它的构造函数、初使化代码、各种JVM安全检查以及其它的一些底层的东西。即使构造函数是私有，我们也可以通过这个方法创建它的实例,对于单例模式，简直是噩梦，哈哈  
83     public native Object allocateInstance(Class cls) throws InstantiationException;  
```

##### 7. 该方法获取对象中offset偏移地址对应的整型field的值,支持volatile load语义
```java
    public native Object getObjectVolatile(Object var1, long var2);

    public native void putObjectVolatile(Object var1, long var2, Object var4);
```

##### 8. `park()和unPark()`使用示例

线程挂起和终止线程挂起方法。

1. `FutureTask`在三种情况下使用`LockSupport`挂起和释放线程：
```java
private int awaitDone(boolean timed, long nanos)方法中：
    1.如果指定了挂起时间： LockSupport.parkNanos(this, nanos);
    2.如果没有指定挂起时间： LockSupport.park(this);

private void finishCompletion()方法中：移除和signal所有等待的线程
    1. 将链表中引用指向null-此时并未释放线程，线程对象仍在阻塞；
    2. 恢复挂起的线程，让其结束执行。
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
```

2. `LockSupport`中三种方法的实现，都是通过调用`Unsafe`的`park(boolean isAbsolutely,long nanoTime)`和`void unPark(Object thread)`
```java
    //恢复线程
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
    
    //挂起线程(绑定挂起的线程和相应的阻塞对象)
    public static void parkNanos(Object blocker, long nanos) {
        if (nanos > 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            UNSAFE.park(false, nanos);
            setBlocker(t, null);
        }
    }
    
    //获取相对偏移量
    private static final long parkBlockerOffset;
    UNSAFE = sun.misc.Unsafe.getUnsafe();
    Class<?> tk = Thread.class;
    parkBlockerOffset = UNSAFE.objectFieldOffset
    (tk.getDeclaredField("parkBlocker"));
    //通过相对偏移量更新Thread的变量parkBlocker
    private static void setBlocker(Thread t, Object arg) {
        // Even though volatile, hotspot doesn't need a write barrier here.
        UNSAFE.putObject(t, parkBlockerOffset, arg);
    }

    //Thread类中变量
    volatile Object parkBlocker;

```