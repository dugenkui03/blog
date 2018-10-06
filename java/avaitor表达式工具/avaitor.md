[注]：使用的话多学习[example](https://github.com/killme2008/aviator/tree/master/src/test/java/com/googlecode/aviator/example);




#### 一.基本简介
Aviator是一个高性能、轻量级的 java 语言实现的表达式求值引擎.

##### 1.1运行方式

其他轻量级的求值器一般都是通过**解释**的方式运行, 而Aviator则是直接**将表达式编译成 JVM 字节码, 交给 JVM 去执行**。

##### 1.2 aviator特性

1. 支持绝大多数运算操作符，包括算术操作符、关系运算符、逻辑操作符、位运算符、正则匹配操作符(=~)、三元表达式(?:)；

2. 支持操作符优先级和括号强制设定优先级；

3. 逻辑运算符支持短路运算；

4. 支持丰富类型，例如nil、整数和浮点数、字符串、正则表达式、日期、变量等，支持自动类型转换；

5. 内置一套强大的常用**函数库**；

6. 可**自定义函数**，易于扩展；

7. 可重载操作符；

8. 支持**大数**运算(BigInteger)和**高精度**运算(BigDecimal)；

9. 性能优秀。

#### 二.使用
**<font color=red>Aviator的入口类是`com.googlecode.aviator.AviatorEvaluator` </font>**，包括编译和执行表达式的方法(**<font color=blue>通过调用`AviatorEvaluatorInstance`单例实例对应的方法</font>**)：

- 获取`AviatorEvaluatorInstance`对象:`AviatorEvaluator`使用的是单例实例：

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
  
  
 /** 如果想再不同的场景使用不同的 求值器，则可通过一下方法获取
   * Create a aviator evaluator instance.
   * <p></p> 创建一个 表达式求值程序实例
   */
  public static AviatorEvaluatorInstance newInstance() {
    return new AviatorEvaluatorInstance();
  }
```
- 编译和执行方法：
```java
  /**
   * Compile a text expression to Expression object
   * <p></p>使用指定的缓存模式编译字符串形式的表达式。<p></p>
   *
   * 编译后的表达式存放在  AviatorEvaluatorInstance 的 ConcurrentHashMap<String, FutureTask<Expression>> cacheExpressions
   *                                = new ConcurrentHashMap<String, FutureTask<Expression>>();
   * 中，通过表达式cacheExpressions.get(expKey).get()获取。
   *
   * @param expression text expression
   * @param cached Whether to cache the compiled result,make true to cache it.
   * @return
   */
  public static Expression compile(final String expression, final boolean cached) {
    /**
     * 1）获取 求值程序实例
     * 2）编译表达式——在 求值程序类中计算
     */
    return getInstance().compile(expression, cached);
  }
  
  
  /**
   * Execute a text expression with environment<p></p>
   * 执行一个表达式，使用指定的环境和缓存策略——如果不指定则默认不绑定参数环境和缓存
   *
   * @param expression text expression
   * @param env Binding variable environment 绑定变量的"环境"
   * @param cached Whether to cache the compiled result,make true to cache it.
   */
  public static Object execute(String expression, Map<String, Object> env, boolean cached) {
    return getInstance().execute(expression, env, cached);
  }

```

##### 2.1 执行表达式&&数字和选项Option类

###### 1）执行简单表达式只要一行：
```java
Long result = (Long) AviatorEvaluator.execute("1+2+3");
```
- 对于数值类型，Aviator只支持Long和Double—Integer和Float会自动转换；
- 可通过`AviatorEvaluator.setOption(Options.XXX, true);`对计算进行配置，比如`    ...Options.ALWAYS_PARSE_FLOATING_POINT_NUMBER_INTO_DECIMAL, true)`会将表达式中的浮点数解析成BigDecimal。其他选项包括 [选项列表说明](https://github.com/killme2008/aviator/wiki/%E5%AE%8C%E6%95%B4%E9%80%89%E9%A1%B9%E5%88%97%E8%A1%A8%E8%AF%B4%E6%98%8E)：  
```java
  /**
   * 计算多行表达式时，是否跟踪计算过程:打开后将在控制台打印整个表达式的求值过程。
   *请勿在生产环境打开，将极大地降低性能。默认为 false 关闭
   */
   TRACE_EVAL,
```

###### 2)三元表达式(if-else)
Aviator 没有提供if else语句,但是提供了三元运算符?:,形式为bool ? exp1: exp2。 其中bool必须为Boolean类型的表达式, 而exp1和exp2可以为任何合法的 Aviator 表达式,并且不要求exp1和exp2返回的结果类型一致。
```java
    public static void main(String[] args) {
        Expression expression=AviatorEvaluator.compile("a>b? a+b:a*b");
        Map<String,Object> env=new HashMap<>();
        env.put("a",2);
        env.put("b",1);
        System.out.println(expression.execute(env));
    }
```
###### 3)重载操作符

类`aviator.lexer.token.OperatorType`包含了Aviator支持的各种运算符，如果程序中有对运算符重载的需要，则可以通过`Aviator.addOpFunction(XX,new AbstractFunction(){call})`进行操作符的重载，也可以使用这个方法添加操作符
```java
    System.out.println(AviatorEvaluator.execute("1+2"));

    new Thread(()->
            AviatorEvaluator.addOpFunction(OperatorType.ADD, new AbstractFunction() {
                @Override
                public AviatorObject call(Map<String, Object> env, AviatorObject arg1, AviatorObject arg2) {
                    return new AviatorString("override");
                }
                //Get the function name:获取函数名称
                @Override
                public String getName() {
                    return "+";
                }
            })
    ).start();

    TimeUnit.SECONDS.sleep(1);
    new Thread(()->System.out.println(AviatorEvaluator.execute("1+2"))).start();

    TimeUnit.SECONDS.sleep(1);
    System.out.println(AviatorEvaluator.newInstance().execute("1+2")); 
    
    output:
        3
        override
        3
```
- 操作符和“操作符说明”放在`AvaitorEvaluatorInstance`中，是其非静态成员变量——**<font color=red>因此除了使用`AviatorEvaluator.newInstance()`方法，所有计算都公用一套操作符解释。</font>**所以第三次输出结果又是3。

###### 4)正则表达式
字符串与正则表达式通过 **`=~`** 操作符来匹配,结果为一个 Boolean 类 型, 因此可以用于三元表达式判断。正则表达式规则跟 Java 完全一样,因为内部其实就是使用java.util.regex.Pattern做编译的
```java
    Map<String,Object> env=new HashMap<>();
    env.put("email","dugk@foxmail.com");
    System.out.println(AviatorEvaluator.execute("email=~ /([\\w0-8]+)@\\w+[\\.\\w+]+/ ? $1:'unknow'", env));

    env.put("email","xxx");
    System.out.println(AviatorEvaluator.execute("email=~ /([\\w0-8]+)@\\w+[\\.\\w+]+/ ? $1:'unknow'", env));
    
    output:
        dugk
        unknow
```
- Aviator 会自动将匹配成功的捕获分组(capturing groups) 放入 env ${num}的变量中,其中$0 指代整个匹配的字符串,而$1表示第一个分组，$2表示第二个分组以此类推；

-  **分组捕获放入 env 是默认开启的，因此如果传入的 env 不是线程安全并且被并发使用，可能存在线程安全的隐患，不同的环境对应不同的Env**。关闭分组匹配，可以通过 `AviatorEvaluator.setOption(Options.PUT_CAPTURING_GROUPS_INTO_ENV, false)`; 来关闭，对性能有稍许好处。下图是程序代码执行到将结果放进环境map对应的结果map(Env继承了Map)代码处的时候：
![image](https://wx1.sinaimg.cn/mw690/006Xp67Kly1fufeb2gyx9j30s80fmq6e.jpg)

##### 2.2 编译表达式：提高性能

以上执行表达式实际上是执行了**编译+执行**的动作。更好的方式是： **<font color=red>先编译一个含有变量的表达式，然后执行含有不同变量的环境。这样性能更高。</font>** 示例如下：

```java
//compile方法可以将表达式编译成Expression的中间对象,
//当要执行表达式的时候传入env并调用Expression的execute方法即可
    public static void main(String[] args){
       String expStr="a*b";
       Expression expression=AviatorEvaluator.compile(expStr);
       Map<String,Object> env=new HashMap<>();//注意一定要是<String,Object>的
       env.put("a",3L);
       env.put("b",2L);

        System.out.println("result:"+expression.execute(env));
    }
```
- 启用缓存：调用`AviatorEvaluator.compile(String,boolean)`第二个参数为true即可
```java
    public static Expression compile(String expression, boolean cached){}
```
- 缓存是一个map数据结构
```java
 /** FutureTask 的 call()方法执行具体的计算；get()方法可以阻塞获取计算结果
   * Compiled Expression cache。操作包含：
   *    1.查询get、清空clear、加入put、移除remove，后三种操作都需要加锁
   *    2.compile中会先查询，后加入；
   *    3.可显式调用清空缓存：防止大对象；
   *
   *    4.使用缓存的话会产生竞争——缓存对象会被锁住而无法被其他线程写入。fixme 但是查询get不需要加锁
   */
  private final ConcurrentHashMap<String, FutureTask<Expression>> cacheExpressions = new ConcurrentHashMap<String, FutureTask<Expression>>();

```
- 移除某个表达式缓存通过如下方法：
```java
    public static void invalidateCache(String expression);
```
##### 2.3 优化级别

运行时优化倾向，有两种选择:
- `AviatorEvaluator.EVAL`，默认值，以**运行时的性能优先**，编译会花费更多时间做优化，目前会做一些**常量折叠、公共变量提取的优化**。适合长期运行的表达式。**<font color=red> 表达式经常不会变化。</font>**
- `AviatorEvaluator.COMPILE` 不会做任何编译优化，牺牲一定的运行性能，适合需要频繁编译表达式的场景,**<font color=red> 比如经常编译不同的表达式。</font>**


设置和获取优化级别的代码如下：
```java
AviatorEvaluator.setOptimize(1);
AviatorEvaluator.getInstance().getOption(Options.OPTIMIZE_LEVEL);
```







