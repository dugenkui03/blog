
##### 1. 介绍

###### 主要方法
`FutureTask`实现了`Runnable`和`Future`接口，其主要方法是：
```java
1. 可以作为任务让线程池(Scheduled)ExecutorService执行；
2. 异步获取(Scheduled)ExecutorService执行任务的返回值，异步获取结果get()/get(time);
3. 取消任务执行。
```
![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fun8venk2bj30r40imgn5.jpg)

###### 状态和`cancle(boolea)`方法
`FutureTask`实现了`Runnable`接口，根据其`run()`方法的执行，可以将其氛围三种状态：
1. 未启动：`run()`方法未执行；
2. 已启动:`run()`方法执行中；
3. 已完成：`run()`方法执行结束。

未启动时，执行`cancle()`则任务永远不会执行；启动时，`cancle(true)`将以 **<font color=red>中断执行任务线程</font>** 的方式中断任务，`cancle(true)`则不会产生任何影响；任务结束时，调用任何`cancle`方法都会返回false；


##### 2. 主要特点和使用场景
 **<font color=red>`FutureTask`可以保证任务只执行一次，并且可以异步获取执行结果，此特点可让其作为缓存被使用。</font>**
1. `FutureTask`在高并发环境下确保**任务只执行一次**：将要执行的任务作为`FutureTask`的参数，然后将`FutureTask`对象给submit(XX)方法执行；
2. 异步获取任务结果；
3. 取消任务。

##### 3.缓存实现(大量的可能重复的耗时计算，缓存对性能的提升是非常高的)
对于性能消耗较大的kv程序—如根据key获取value耗时较长，我们希望将kv存放在map中缓存。如[`Aviator`](https://github.com/dugenkui03/aviator/wiki)

下边我们写一个程序用于求值程序的缓存(求一个数的立方并放进value)：

```java
/**
 * 返回构造器参数的平方
 */
class Calcutor implements Callable<Integer>{
    private Integer factor;

    public Calcutor(Integer factor) {
        this.factor = factor;
    }

    @Override
    public Integer call() throws Exception {
        //模拟计算耗时
        TimeUnit.NANOSECONDS.sleep(100000000);
        return factor*factor;
    }
}

public class CacheTest {
    static private final Map<Integer, FutureTask<Integer>> squareCache=new ConcurrentHashMap();

    /**
     * 计算某数的立方
     * @param cache 是否使用缓存，使用的话表示：查询缓存尝试从缓存获取；将Integer对应的
     *
     *     1. 查看是否存在缓存中，存在则阻塞获取任务结果；
     *     2. 不存在缓存中，则构造任务，并使用putIfAbsent放进缓存；
     *     3. 如果是第一次放进缓存map，即putIfAbsent返回null，则执行任务，并返回结果(第一个步骤没有，但是第二个步骤可能是几个线程并行执行）；
     */
    static Integer cal(final Integer fac,final boolean cache) throws Exception {
        if(cache){
            //查看是否在缓存中，有则阻塞获取计算结果
            FutureTask<Integer> calTask=squareCache.get(fac);
            if(calTask!=null){
                return calTask.get();
            }

            //不再缓存中则构造任务(可能多个线程并发执行),并放进缓存中，然后(如果是第一次放进)执行并返回结果
            //fixme 如果有并发线程也构造了任务，并尝试放进map缓存，则返回值不为null，直接跳过if中的计算逻辑，异步获取计算结果
            calTask=new FutureTask<>(new Calcutor(fac));
            FutureTask<Integer> exitedTask=squareCache.putIfAbsent(fac,calTask);
            if(exitedTask==null){
                exitedTask=calTask;
                exitedTask.run();
            }
            return calTask.get();
        }else{
            //可改进：将计算过程抽象出去使用多线程调用
            return new Calcutor(fac).call();
        }
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executorService=Executors.newFixedThreadPool(1);

        long t1=System.currentTimeMillis();
        Random random=new Random(100);
        for (int i = 0; i < 100; i++) {
            int fac=random.nextInt(30);
            System.out.println(i+":"+cal(fac, true));
        }
        System.out.println("cost:"+(System.currentTimeMillis()-t1));
    }
}

```

##### 4.`Aviator`中的缓存

```java

  public Expression compile(final String expression, final boolean cached) {
    /**
     * 如果编译的表达式为空，则返回异常
     */
    if (expression == null || expression.trim().length() == 0) {
      throw new CompileExpressionErrorException("Blank expression");
    }
    if (cached) {
      /**
       * FureTask带有返回值的任务
       * 查询缓存，如果缓存不为空，则直接返回结果
       * ConcurrentHashMap <String, FutureTask<Expression>> task
       */
      FutureTask<Expression> task = cacheExpressions.get(expression);
      //如果已经放在缓存中，异步请求结果即可
      if (task != null) {
        return getCompiledExpression(expression, task);
      }
      /**
       * 如果缓存为空的话，进行编译，然后把结果放进缓存
       */
      task = new FutureTask<Expression>(new Callable<Expression>() {
        @Override
        public Expression call() throws Exception {
          return innerCompile(expression, cached);
        }

      });
      //fixme  缓存在这里放进去的并执行：如果之前没有表达式expression对应的task——即没有缓存的内容，则返回null
      //使用putIfAbsent是因为如果expression有对应的任务，必定相同——在kv值放进缓存之前并发进入的同一表达式的编译请求可以不用在浪费性能替换相同的value。详见笔记《aviator注意点一》
      FutureTask<Expression> existedTask = cacheExpressions.putIfAbsent(expression, task);
      //如果是第一次放进缓存(putIfAbsent返回已有值或者第一次放进去则返回null)
      if (existedTask == null) {
        existedTask = task;
        //fixme 任务在这里执行，注意不是使用线程池，而是串行执行后获取
        existedTask.run();
      }
      return getCompiledExpression(expression, existedTask);
    }
    /**
     * 若不使用缓存：使用当前线程的类加载器
     */
    else {
      return innerCompile(expression, cached);
    }

  }
  
    /**
   * 异步阻塞获取编译后的表达式
   */
  private Expression getCompiledExpression(final String expression, FutureTask<Expression> task) {
    try {
      //阻塞获取结果
      return task.get();
    } catch (Exception e) {
      cacheExpressions.remove(expression);
      throw new CompileExpressionErrorException("Compile expression failure:" + expression, e);
    }
  }
  
    /**
   * 编译表达式
   * @param expression 被编译的表达式字符串
   * @param cached 是否使用缓存：todo cache不仅是是否将结果放在map？这里有什么用
   */
  private Expression innerCompile(final String expression, boolean cached) {
    //新建词法分析器
    ExpressionLexer lexer = new ExpressionLexer(this, expression);
    /**
     * 生成一个成员变量包含类加载器的"代码生成器" fixme 内部代码有根据 缓存策略 获取类加载器
     */
    CodeGenerator codeGenerator = newCodeGenerator(cached);
    /**
     *  表达式解析程序：
     *      包含被解析的表达式字串
     *      包含自定义类加载器
     */
    ExpressionParser parser = new ExpressionParser(this, lexer, codeGenerator);
    /**
     * fixme 开始解析表达式
     */
    Expression exp = parser.parse();
    if ((boolean) getOption(Options.TRACE_EVAL)) {
      ((BaseExpression) exp).setExpression(expression);
    }
    return exp;
  }
```