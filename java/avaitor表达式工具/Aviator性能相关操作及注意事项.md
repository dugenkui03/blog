

#### 一.性能相关

##### 1.1 优化级别

Aviator编译后才执行，对应有两种优化级别：
- `AviatorEvaluator.EVAL`，默认值，会花较多时间在优化编译上。**<font color=red> 适合长期运行的表达式</font>**
- `AviatorEvaluator.COMPILE` 不会做任何编译优化，适合 **<font color=red> 需要频繁编译表达式的场景。</font>**

优化级别的设置和查询如下：
```java
AviatorEvaluator.setOptimize(1);
AviatorEvaluator.getInstance().getOption(Options.OPTIMIZE_LEVEL);
```
##### 1.2 多行表达式 打印计算过程

对于多行表达式(;分割)，最后返回的结果是最后一个表达式的值。需要查看前边表达式的值则可以设置打印计算过程来查看。<font color=red>**但会极大降低性能**</font>：
```java
Options.TRACE_EVAL
```

##### 1.3 正则表达式
对于正则表达式，avaitor会将与模式匹配的字符串和捕获的分组放进一个map中。如果不需要捕获的分组，则可以通过**设置`AviatorEvaluator.setOption(Options.PUT_CAPTURING_GROUPS_INTO_ENV,false)`来稍提高性能。**

而且如果设置环境的map不是线程安全的，但是在并发环境下使用，则可能存在线程隐患。Env初始化代码：

```java

//第一ClassExpression extends BaseExpression：设置包含环境和捕获分组的Env类实例
//map为环境参数，比如表达式a+b中的(a,1),(b,2)。正则表达式中则是(str1,exp1)\(str2,exp2)
public Object execute(Map<String, Object> map){
    if (map == null) {
      map = Collections.emptyMap();
    }
    Env env = newEnv(map);
    ......

//第二BaseExpression：  
protected Env newEnv(Map<String, Object> map) {
    Env env = new Env(map);
    env.setInstance(this.instance);//设置执行表达式实例的求值程序实例
    return env;
}

//第三:Env构造函数
  /** Default values map. */
  private final Map<String, Object> mDefaults;
  
  public Env(Map<String, Object> defaults) {
    mDefaults = defaults;
  }

//第四，Evn：设置执行表达式实例的求值程序实例
  /**
   * Current evaluator instance that executes current expression.
   */
  private AviatorEvaluatorInstance instance;
  
  public void setInstance(AviatorEvaluatorInstance instance) {
    this.instance = instance;
  }

```

##### 1.4 重载运算符有一定的性能损失
`AviatorEvaluator.addOpFunction(opType, func)`不仅可以重载运算符，而且可以自定义运算符。运算符及其解释放在`IdentityHashMap`中。



##### 1.5 编译表达式
执行表达式`AviatorEvaluator.execute(...)`实际上经过了编译和执行两个步骤。可以通过先编译一个通用的表达式，然后通过传入不同的环境来复用表达式，以此来提升性能:
```java
Expression exp=AviatorEvaluator.compile("a+b");
Map<String,Object> map=new HashMap();
map.put("a",1);
exp.execute(env);
...


性能好于：
AviatorEvaluator.execute(1+2);
AviatorEvaluator.execute(2+2);
AviatorEvaluator.execute(3+2);
```

##### 1.6 缓存策略

编译和执行表达式默认是不对表达式进行缓存的，但是可以通过调用含有缓存参数的方法来将结果缓存。

```java
AviatorEvaluator方法：

public static Expression compile(final String expression, final boolean cached)

public static Object execute(String expression, Map<String, Object> env, boolean cached)；
```

##### 1.7 加载类 3.2.0版本升级

3.2.0版本升级之后优先使用 `Class<?>  sun.misc.Unsafe#defineAnonymousClass`加载类，编译性能更好（没有 ClassLoader **加载校验**等环节），表达式编译后的<font color=red>**匿名类将可以被 GC 正常回收，解决在编译大量动态表达式的时候导致的内存消耗膨胀问题。**</font>



#### 二.可能会改变内存负载的地方

##### 2.1 缓存

Aviator将缓存的结果放在`ConcurrentHashMap<String, FutureTask<Expression>>`中，相同的表达式不用在加载卸载，而见小了元空间Metadata的负担

```java
/** 
   * Compiled Expression cache。操作包含：
   *    1.
   *    2.compile中会先查询，后加入；
   *    3.可显式调用清空缓存：防止大对象；
   *
   *    4.使用缓存的话会产生竞争——缓存对象会被锁住而无法被其他线程写入。fixme 但是查询get不需要加锁
   */
  private final ConcurrentHashMap<String, FutureTask<Expression>> cacheExpressions = new ConcurrentHashMap<String, FutureTask<Expression>>();
```
- 源码中设计 查询get、加入put、清空clear、移除remove，后三种操作都需要加锁。前两种每次在编译表达式时都可能调用，后两种则显式调用；
- 对于不再需要的表达式，可以通过`void invalidateCache(String expression) `方法移除其缓存；
- 对于可能出现的大对象，可使用clear清空缓存——实验平台不可能出现这种情况；
- **<font color=red>使用缓存的时候所有类默认使用同一个类加载器卸载，因此加载总量很大的话也会导致元空间溢出**</font>。关于元空间设置，参考3.4小节：

##### 2.2 求值程序实例 

`AviatorEvaluator`编译或执行表达式，都是通过持有的 `AviatorEvaluatorInstance` 类来计算的
```java
    /**
     * 单例 静态的求值程序实例
     */
  private static class StaticHolder {
    private static AviatorEvaluatorInstance INSTANCE = new AviatorEvaluatorInstance();
  }
  
/**
 * 获取单例 的求值程序实例
 */
  public static AviatorEvaluatorInstance getInstance() {
    return StaticHolder.INSTANCE;
  }
```
默认情况下都是使用的此静态实例。但是对于<font color=red>**需要不同的缓存策略和操作符解释的不同场景**</font>，可以使用如下方法获取多个实例：
```java
  public static AviatorEvaluatorInstance newInstance() {
    return new AviatorEvaluatorInstance();
  }
```
**此项没必要优化**，因为用户都需要缓存(一次编译、多次获取)、并且表达式总量也不大，可以放到同一个map中统一管理——map最大容量是Integer.MAX_VALUE。

##### 2.3 元空间

Aviator使用缓存策略的时候，所有表达式使用同一个类加载器加载。**如果不同表达式的量特别大，则可能导致元空间溢出**。
```java
  //AviatorEvaluatorInstance
  public AviatorClassLoader getAviatorClassLoader(boolean cached) {
    if (cached) {
      return aviatorClassLoader;
    } else {
      return new AviatorClassLoader(Thread.currentThread().getContextClassLoader());
    }
  }
```

###### 1）加大元空间永久代
可以通过使用元空间默认值(可使用的物理内存大小)来解决元空间溢出。java7的话可以通过`-XX:PermSize=256m -XX:MaxPermSize=256m`来设置永久带大小。

###### 2)类加载器

类加载器未被回收则其加载的类也不能被回收(且类的Class对象不能被引用，对象全部被回收)。
1. 不同的表达式不同的缓存策略；
2. 使用不同类加载器加载不同的表达式：**不同的AviatorEvaluatorInstance实例使用不同的类加载器**
```java
  /**AviatorEvaluatorInstance
   */
  private AviatorClassLoader aviatorClassLoader;
  {
      aviatorClassLoader = AccessController.doPrivileged(new PrivilegedAction<AviatorClassLoader>() {
          @Override
          public AviatorClassLoader run() {
              return new AviatorClassLoader(AviatorEvaluatorInstance.class.getClassLoader());
          }
      });
    }
```

#### 三.优化思路

##### 3.1 缓存
表达式编译缓存；done

##### 3.2 优化级别
优化级别增加开关——一般来说编译的表达式需要多次重复执行，使用默认优化级别就好。

##### 3.3.在接口中增加相应的参数给业务方，让其根据自身情况设置(当前项目只使用了compile(String exp,boolean cache)方法）

##### 3.4 参看2.2\2.3，是否对不同的场景(业务方)使用不同的求值器实例。


#### 四.其他
元空间、类加载器介绍。

##### 4.1 元空间
元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：**元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制**，但可以通过以下参数来指定元空间的大小和GC设置：

- -XX:MetaspaceSize，初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值。
- -XX:MaxMetaspaceSize，最大空间，默认是没有限制的。
- -XX:MinMetaspaceFreeRatio，在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集
- -XX:MaxMetaspaceFreeRatio，在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集。
　　
##### 4.2 类加载器

**不使用缓存时**，有两个`AviatorClassLoader`实例，其中一个时GC Root，即无法被回收，因此其加载的`Expression`也无法被回收。可知另外一个`AviatorClassLoader`实例负责加载`Expression`。dump文件如下：

![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fugn8lrzzzj31720eutbs.jpg)

- 其中一个被`AMSCodeGenerator和AviatorClassloader`引用，另一个被`AviatorEvaluatorInstance`引用；
- 使用缓存时，两个类加载器的来源如下：
```java

  /**
   * Returns classloader返回类加载器
   *  1.使用缓存则放回当前this.类加载器变量对象
   *  2.否则则新建类加载器，父类加载器是加载当前线程上下文的类加载器
   *
   */
  public AviatorClassLoader getAviatorClassLoader(boolean cached) {
    if (cached) {
      return aviatorClassLoader;
    } else {
      return new AviatorClassLoader(Thread.currentThread().getContextClassLoader());
    }
  }
  
    private AviatorClassLoader aviatorClassLoader;
  //初始化类加载器
  {
      aviatorClassLoader = AccessController.doPrivileged(new PrivilegedAction<AviatorClassLoader>() {
          @Override
          public AviatorClassLoader run() {
              return new AviatorClassLoader(AviatorEvaluatorInstance.class.getClassLoader());
          }
      });
    }

```
使用缓存时，只有一个`AviatorClassLoader`实例，且不是GC Root，被`AviatorEvaluatorInstance`引用。来源于初始化`AviatorClassLoader`实例时：

```java
    private AviatorClassLoader aviatorClassLoader;
  //初始化类加载器
  {
      aviatorClassLoader = AccessController.doPrivileged(new PrivilegedAction<AviatorClassLoader>() {
          @Override
          public AviatorClassLoader run() {
              return new AviatorClassLoader(AviatorEvaluatorInstance.class.getClassLoader());
          }
      });
    }
```

todo

```java
public class ClassLoaderTest {

    public static void main(String[] args) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        ClassLoader myLoader = new ClassLoader() {
            //output:false
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String className = name.substring(name.lastIndexOf(".") + 1) + ".class";

                    //返回读取指定资源的输入流
                    InputStream is = getClass().getResourceAsStream(className);
                    if (is == null) return super.loadClass(name);
                    byte[] b = new byte[is.available()];
                    is.read(b);

                    //将一个byte数组转换为Class类的实例
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
            //output:true
            public Class<?> defineClass(String name, byte[] b) {
                return defineClass(name, b, 0, b.length);
            }
        };

        Object object = myLoader.loadClass("com.googlecode.aviator.ClassLoaderTest").newInstance();
        System.out.println(object instanceof com.googlecode.aviator.ClassLoaderTest);
    }
}
```

##### 参考文献
1. [元空间介绍](https://www.cnblogs.com/paddix/p/5309550.html)
2. [aviator使用手册](https://github.com/killme2008/aviator/wiki)
3. [4.0.1版本源码说明测试](https://github.com/dugenkui03/aviator/tree/41-dev)