#### `10^6^纳秒=10^3^微秒=1毫秒`


##### 1. 枚举使用
 **<font color=red>这种在枚举外调用枚举中声明的方法的设计思路很有意思，也算是动态调用？</font>**

1. 定义抽象方法doWork()，可在具体枚举值中实现<font color=red>不同枚举值进行不同的处理;</font>
2. 定义调用参数方法invokeWorker()，调用不同枚举值的抽象方法的实现；


demo如下：
```java
public class EnumTest {
    enum EnumDemo{
        DEMOX("demoX"){
            @Override
            void enumFunc(){
                System.out.println(getName());
            }
        },
        DEMOY("demoY"){
            @Override
            void enumFunc(){
                System.out.println(getName());
            }
        };

        abstract void enumFunc();

        public void invokeEnumFunc(){
            enumFunc();
        }

        private String name;
        EnumDemo(String name){this.name=name;}
        public String getName() {
            return name;
        }
    }

    public static void main(String[] args) {
        EnumDemo.DEMOX.invokeEnumFunc();
        EnumDemo.DEMOY.invokeEnumFunc();
    }
}

output:
        demoX
        demoY

```

##### 2.源码讲解

1. 所有时间单位都是一个包含转换方法的枚举，进率保存常量中；
2. 大时间单位转小时间单位的时候会判断一下是否溢出， **<font color=red>需要判定上下界，因为基数为负则可能下界溢出</font>。**
3. TimeUnit.时间单位SECOND/NANOSECONDS.sleep(10^n^)的调用栈；

```
public enum TimeUnit {

    // Handy constants for conversion methods
    static final long C0 = 1L;//纳秒
    static final long C1 = C0 * 1000L;//纳秒-》微秒
    static final long C2 = C1 * 1000L;//纳秒-》毫秒
    static final long C3 = C2 * 1000L;//纳秒-》秒
    static final long C4 = C3 * 60L;//纳秒-》分钟
    static final long C5 = C4 * 60L;//纳秒-》小时
    static final long C6 = C5 * 24L;//纳秒-》天

    static final long MAX = Long.MAX_VALUE;   
   
    /**
     * 纳秒
     */
    NANOSECONDS {
        public long toNanos(long d)   { return d; }
        public long toMicros(long d)  { return d/(C1/C0); }
        public long toMillis(long d)  { return d/(C2/C0); }
        public long toSeconds(long d) { return d/(C3/C0); }
        public long toMinutes(long d) { return d/(C4/C0); }
        public long toHours(long d)   { return d/(C5/C0); }
        public long toDays(long d)    { return d/(C6/C0); }
        public long convert(long d, TimeUnit u) { return u.toNanos(d); }
        int excessNanos(long d, long m) { return (int)(d - (m*C2)); }
    },

    /**
     * Time unit representing one thousandth of a millisecond
     */
    MICROSECONDS {
        public long toNanos(long d)   { return x(d, C1/C0, MAX/(C1/C0)); }
        public long toMicros(long d)  { return d; }
        public long toMillis(long d)  { return d/(C2/C1); }
        public long toSeconds(long d) { return d/(C3/C1); }
        public long toMinutes(long d) { return d/(C4/C1); }
        public long toHours(long d)   { return d/(C5/C1); }
        public long toDays(long d)    { return d/(C6/C1); }
        public long convert(long d, TimeUnit u) { return u.toMicros(d); }
        int excessNanos(long d, long m) { return (int)((d*C1) - (m*C2)); }
    },

    /**
     * Time unit representing one thousandth of a second
     */
    MILLISECONDS {
        public long toNanos(long d)   { return x(d, C2/C0, MAX/(C2/C0)); }
        public long toMicros(long d)  { return x(d, C2/C1, MAX/(C2/C1)); }
        public long toMillis(long d)  { return d; }
        public long toSeconds(long d) { return d/(C3/C2); }
        public long toMinutes(long d) { return d/(C4/C2); }
        public long toHours(long d)   { return d/(C5/C2); }
        public long toDays(long d)    { return d/(C6/C2); }
        public long convert(long d, TimeUnit u) { return u.toMillis(d); }
        int excessNanos(long d, long m) { return 0; }
    },

    /**
     * Time unit representing one second
     */
    SECONDS {
        public long toNanos(long d)   { return x(d, C3/C0, MAX/(C3/C0)); }
        public long toMicros(long d)  { return x(d, C3/C1, MAX/(C3/C1)); }
        public long toMillis(long d)  { return x(d, C3/C2, MAX/(C3/C2)); }
        public long toSeconds(long d) { return d; }
        public long toMinutes(long d) { return d/(C4/C3); }
        public long toHours(long d)   { return d/(C5/C3); }
        public long toDays(long d)    { return d/(C6/C3); }
        public long convert(long d, TimeUnit u) { return u.toSeconds(d); }
        int excessNanos(long d, long m) { return 0; }
    },

    /**
     * Time unit representing sixty seconds
     */
    MINUTES {
        public long toNanos(long d)   { return x(d, C4/C0, MAX/(C4/C0)); }
        public long toMicros(long d)  { return x(d, C4/C1, MAX/(C4/C1)); }
        public long toMillis(long d)  { return x(d, C4/C2, MAX/(C4/C2)); }
        public long toSeconds(long d) { return x(d, C4/C3, MAX/(C4/C3)); }
        public long toMinutes(long d) { return d; }
        public long toHours(long d)   { return d/(C5/C4); }
        public long toDays(long d)    { return d/(C6/C4); }
        public long convert(long d, TimeUnit u) { return u.toMinutes(d); }
        int excessNanos(long d, long m) { return 0; }
    },

    /**
     * Time unit representing sixty minutes
     */
    HOURS {
        public long toNanos(long d)   { return x(d, C5/C0, MAX/(C5/C0)); }
        public long toMicros(long d)  { return x(d, C5/C1, MAX/(C5/C1)); }
        public long toMillis(long d)  { return x(d, C5/C2, MAX/(C5/C2)); }
        public long toSeconds(long d) { return x(d, C5/C3, MAX/(C5/C3)); }
        public long toMinutes(long d) { return x(d, C5/C4, MAX/(C5/C4)); }
        public long toHours(long d)   { return d; }
        public long toDays(long d)    { return d/(C6/C5); }
        public long convert(long d, TimeUnit u) { return u.toHours(d); }
        int excessNanos(long d, long m) { return 0; }
    },

    /**
     * Time unit representing twenty four hours
     */
    DAYS {
        public long toNanos(long d)   { return x(d, C6/C0, MAX/(C6/C0)); }
        public long toMicros(long d)  { return x(d, C6/C1, MAX/(C6/C1)); }
        public long toMillis(long d)  { return x(d, C6/C2, MAX/(C6/C2)); }
        public long toSeconds(long d) { return x(d, C6/C3, MAX/(C6/C3)); }
        public long toMinutes(long d) { return x(d, C6/C4, MAX/(C6/C4)); }
        public long toHours(long d)   { return x(d, C6/C5, MAX/(C6/C5)); }
        public long toDays(long d)    { return d; }
        public long convert(long d, TimeUnit u) { return u.toDays(d); }
        int excessNanos(long d, long m) { return 0; }
    };



    /**
     * Scale d by m, checking for overflow:
     *校验转换成更小的时间单位是，返回值是否会溢出：溢出则返回上下界
     * @param d 被转换值，可为负，所以需要判定下界
     * @param m 转换成更小的单位时需要想乘的因子
     * @param over 保证d*m不溢出的上下限
     */
    static long x(long d, long m, long over) {
        if (d >  over) return Long.MAX_VALUE;
        //注意：因为负数乘以因子后会接近与Long.MIN_VALUE
        if (d < -over) return Long.MIN_VALUE;
        return d * m;
    }
    
    //TimeUnit.XXX.sleep调用此方法，如果时间单位比毫秒小，四舍五入(在sleep方法中)
    public void sleep(long timeout) throws InterruptedException {
        if (timeout > 0) {
            long ms = toMillis(timeout);
            //纳秒：int excessNanos(long d, long m) { return (int)(d - (m*C2)); }
            //微秒：int excessNanos(long d, long m) { return (int)((d*C1) - (m*C2)); }
            //excessNanos用于求timeout转换成毫秒后的纳秒余数，用于休眠时四舍五入
            int ns = excessNanos(timeout, ms);
            Thread.sleep(ms, ns);
        }
    }
    
    //Thread中的类：睡眠指定时间，nano四舍五入加在Millis上。如果超范围会抛异常
    public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }
        sleep(millis);//native方法
    }

    

    /**
     * 用于计算wait、sleep和join时，多余的纳秒的工具类。
     * Utility to compute the excess-nanosecond argument to wait,
     * sleep, join.
     * @param d the duration
     * @param m the number of milliseconds
     * @return the number of nanoseconds
     */
    abstract int excessNanos(long d, long m);

    /**
     * Performs a timed {@link Object#wait(long, int) Object.wait}
     * using this time unit.
     * This is a convenience method that converts timeout arguments
     * into the form required by the {@code Object.wait} method.
     *
     * <p>For example, you could implement a blocking {@code poll}
     * method (see {@link BlockingQueue#poll BlockingQueue.poll})
     * using:
     *
     *  <pre> {@code
     * public synchronized Object poll(long timeout, TimeUnit unit)
     *     throws InterruptedException {
     *   while (empty) {
     *     unit.timedWait(this, timeout);
     *     ...
     *   }
     * }}</pre>
     *
     * @param obj the object to wait on
     * @param timeout the maximum time to wait. If less than
     * or equal to zero, do not wait at all.
     * @throws InterruptedException if interrupted while waiting
     */
    public void timedWait(Object obj, long timeout)
            throws InterruptedException {
        if (timeout > 0) {
            long ms = toMillis(timeout);
            int ns = excessNanos(timeout, ms);
            obj.wait(ms, ns);
        }
    }

    /**
     * Performs a timed {@link Thread#join(long, int) Thread.join}
     * using this time unit.
     * This is a convenience method that converts time arguments into the
     * form required by the {@code Thread.join} method.
     *
     * @param thread the thread to wait for
     * @param timeout the maximum time to wait. If less than
     * or equal to zero, do not wait at all.
     * @throws InterruptedException if interrupted while waiting
     */
    public void timedJoin(Thread thread, long timeout)
            throws InterruptedException {
        if (timeout > 0) {
            long ms = toMillis(timeout);
            int ns = excessNanos(timeout, ms);
            thread.join(ms, ns);
        }
    }
}
```