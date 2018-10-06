
使用场景、注意点

##### 1 使用场景

###### 1.1 使用`ThreadLocal`维持线程封闭性

`ThreadLocal`对象通常用于防止对可变的**单实例变量singleton**或**全局变量**进行共享。因为某个线程对变量的更改会影响到其他线程，通过 **`ThreadLocal`可以为每个使用变量的线程保存一份独立的副本**。`Aviator`中`RandomFunction`类示例：

```java
  private static ThreadLocal<Random> randomLocal = new ThreadLocal<Random>() {
    @Override
    protected Random initialValue() {
      //SecureRandom在算法上对Random进行了改进，并且“没有seed相同则生成随机数相同”这种弊端
      return new SecureRandom();
    }
  };

  @Override
  public AviatorObject call(Map<String, Object> env) {
    return AviatorDouble.valueOf(randomLocal.get().nextDouble());
  }
```

###### 1.2 当某个频繁执行的操作需要一个临时对象，例如缓存区缓存`toString()`操作的结果，而又希望避免在每次执行时都重新分配该临时对象(比如`toString()`操作开销很大），则可以使用该技术


##### 2 注意点

###### 2.1 初次调用`get()`方法时会调用`initialValue`来获取初始值，否则读取值

1. 为某个变量创建`ThreadLocal`对象时创建重写`initialValue()`方法；
2. 在某个线程中调用`ThreadLocal`对象的`get()`时：
    1. 如果第一次在此线程调用`get`方法，则此线程`ThreadLocal.ThreadLocalMap`对象在`setInitialValue`中被初始化；
    2. **如果此线程已经调用过某个`ThreadLocal`对象，则其`ThreadLocal.ThreadLocalMap`变量不为null，<font color=red>如果其此map变量中也有此ThreadLocal的key，则获取其值**</font>;

```java
   /**
     * 返回调用对象(ThreadLocal)中的当前线程对应的副本值，
     * 如果为null通过调用setInitialValue()方法设初始值并返回
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        /**
         * 1.首先获取当前线程
         */
        Thread t = Thread.currentThread();

        /**
         * 2.根据当前线程获取map;
         */
        ThreadLocalMap map = getMap(t);
        /**
         * 3.以ThreadLocal的引用作为key来在Map中获取对应的value e，否则转到5;
         * fixme 在某个线程中第一次调用ThreadLocal对象的get()方法时，map一定为null
         */
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            /**
             * 4.如果e不为null，则返回e.value
             */
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        /**
         * 5.Map为空或者e为空，则通过initialValue函数获取初始值value.
         */
        return setInitialValue();
    }
    
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     * <p></p>
     * 建立初始值的set()方法的变体，不是用set()方法是因为set()方法可能被重写
     *
     * @return the initial value 返回初始值
     */
    private T setInitialValue() {
        /**
         * initialValue() 默认返回null，是创建ThreadLocal对象的时候需要重写的方法
         */
        T value = initialValue();
        /**
         * 获取当前线程的实例变量(此实例变量只在jdk此类的createMap()方法中被写)
         *    ThreadLocal.ThreadLocalMap threadLocals = null;
         *    return t.threadLocals;
         */
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        /**
         * 如果当前线程的类变量 ThreadLocal.ThreadLocalMap 
         *      不为null，则设置key对应的value：key-当前ThreadLocal对象；value-初始值(某变量副本);
         *      如果为null，则为此线程变量设置初值：key和value同上： t.threadLocals = new ThreadLocalMap(this, firstValue);
         */
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    
    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *  <p></p>
     *  创建与当前ThreadLocal对象关联的ThreawdLocalMap对象，在InheritableThreadLocal中被重写
     *
     * @param t the current thread 当前对象
     * @param firstValue value for the initial entry of the map。map中存放的初始值
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
###### 2.2 数据存放形式

<font color=blue>**`ThreadLocal`可以在线程中存放某个数据的副本**——其形式是在线程变量`ThreadLocal.ThreadLocalMap`中存放键值对—key为ThreadLocal对象，value为ThreadLocal保存的值的副本。</font>

当我们在某个线程中调用`ThreadLocal`对象的`get()`方法时，其实是以自己为key，在当前线程中查找是否有对应的value，无则初始化并返回，有则返回。

```java
    /**
     * 返回调用对象(ThreadLocal)中的当前线程对应的副本值，
     * 如果为null通过调用setInitialValue()方法设初始值并返回
     */
    public T get() {
        //1.首先获取当前线程
        Thread t = Thread.currentThread();

        //根据当前线程获取map，代码可用下行代替(原代码：ThreadLocalMap map = getMap(t)；)
        ThreadLocalMap map=t.threadLocals; //ThreadLocal.ThreadLocalMap;

        //如果当前线程调用了任何一个ThreadLocal对象的get()方法，则线程变量threadLocals不为null；
        if (map != null) {
            //如果当前线程存放了此对象对应的kv值，则直接返回value，否则进行初始化
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
 
        //当前线程中没有调用过此ThreadLocal的get()方法是，会调用下行代码
        return setInitialValue();
    }
    

//数据保存结构如下
Thread类中:
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
ThreadLocal.ThreadLocalMap.Entry结构:
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);//this.referent = referent; key保存的是ThreadLocal对象的引用
                value = v;
            }
        }
```