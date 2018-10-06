
##### 1. `Runnable`和`Callable<v>`、`Future`及其实现类`FutureTask`对比

`Runnable`和`Callable<v>`都是任务的抽象类，不同的是前者不会返回值，后者有返回值。两者源码如下：

- `Runnable`
```
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

- `Callable<V>`
```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

`Future`及其实现类`FutureTask`<font color=red>**用来获取异步计算结果的，说白了就是对具体的Runnable或者Callable对象任务执行的结果进行获取`get() get(x,y)`,取消`cancel()`,判断是否完成`isDone()`和是否取消`isCancelled()`等操作**</font>。其源码如下：
```java

public interface Future<V> {

    //尝试取消任务的执行，如果任务已经完成或者取消，则会尝试将会失败。
    //如果参数是true，表示正在执行的任务可以中断，
    //此方法执行后，isDone总会返回true；如果返回true则isCancelled返回true；
    boolean cancel(boolean mayInterruptIfRunning);

    //如果任务在完成之前取消，返回true
    boolean isCancelled();

    //任务完成则返回TRUE，完成指正常结束、异常或者取消；
    boolean isDone();

    //阻塞获取计算结果(计算未完成则等待)
    V get() throws InterruptedException, ExecutionException;

    //在指定的时间内阻塞获取计算结果
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

可以**作为线程池`ThreadPoolExecutor`的方法`public <T> Future<T> submit(Callable<T> task)`的返回值，其`get()`方法可以阻塞获取`Callable的call()`计算结果。**


###### `FutureTask`
`FutureTask`是`RunnableFuture<v>`的实现类，而`RunnableFuture<V>`是`Runnable`和`Future`的子接口(接口可以多继承)。**注意`FutureTask`实现了`Runnable`接口。

##### 2. 使用示范

展示使用`Callable<V>`+`Future<V>`和`Callable<V>`+`FutureTask<V>`提交有返回值的任务，并且异步获取返回值。

- `Callable<V>`+`Future<V>`：提交`Callable`并获取`Future`类型的返回结果；
- `Callable<V>`+`FutureTask<V>`:将`Callable`作为构造器参数，将自己并提交执行，然后使用其`get`获取结果。



`CallableTask`

```java
public class CallableTask implements Callable<Integer> {
    @Override
    public Integer call() throws InterruptedException {
        System.out.println("开始执行任务");
        TimeUnit.SECONDS.sleep(1);
        int sum=0;
        for (int i = 0; i < 100; i++) {
            sum+=i;
        }
        System.out.println("结束任务执行");
        return new Integer(sum);
    }
}
```

`FutureDemo`
```java
public class FutureDemo {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executor= Executors.newCachedThreadPool();

        //fixme 提交Callable并获取Future类型的返回结果
        CallableTask task=new CallableTask();
        Future<Integer> res=executor.submit(task);

        executor.shutdown();
        System.out.println("计算结果："+res.get());
    }
}
```

`FutureTaskDemo`
```java
public class FutureTaskDemo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executor= Executors.newCachedThreadPool();

        //fixme 不是直接提交Callable，而是将Callable作为构造器参数，将自己并提交执行，然后使用其get获取结果
        CallableTask callableTask=new CallableTask();
        FutureTask futureTask=new FutureTask(callableTask);
        executor.submit(futureTask);

        executor.shutdown();
        System.out.println("计算结果:"+futureTask.get());
    }
}
```

output:
```
开始执行任务
结束任务执行
计算结果:4950
```