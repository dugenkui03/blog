
##### 1. 引言

在多线程环境下，**使用**和**共享**对象时有四种常用策略：
1. <font color=red>**线程封闭**</font>：线程封闭的对象只能由一个线程拥有，线程封闭在线程中，并且只能由这个线程修改。实现技术有==栈封闭==和 ==`ThreadLocal`== 类；
2. 只读共享：只允许读取的且不可修改的对象可以由多个线程安全的并发访问。不可变对象和事实不可变对象此类策略的实现技术；
3. 保护对象：使用特定的锁访问对象；
4. 线程安全的共享：在对象内部实现同步，并且只使用共有接口提供访问对象状态——不太理解。


##### 2. `ThreadLocal`基本思想

实现线程封闭的方式之一是**栈封闭**，只能通过局部变量才能访问对象，而局部变量是封闭在执行的线程中的，其他线程无法修改。另一片博文展开讲解。

###### 优点

`ThreadLocal`是**为每个使用变量的线程都存有一个独立的副本**，因此每个线程对此变量的修改都是都只是在自己工作内存而已，也没有同步内存的时间开销。因此也可以说<font color=red>**栈封闭是用空间换时间和安全性。**</font>


###### 使用场景

**如果对象的分配开销特别高或者在线程中执行的频率特别高，则应该使用ThreadLocal。**


##### 3. 实现

###### 主要方法
- `T iniinitialValue()`：初始化当前线程副本值；
- ` T get()`：返回此线程中thread-local变量副本值，如果当前线程没有对应副本值，则会调用`iniinitialValue`方法返回；
- `set(T value)`：设置当前线程副本值；
- `remove()`：移除当前线程局部变量副本值。


**示例：**
```
public class ThreadLocalDemo implements Runnable {

    private static ThreadLocal<Integer> threadLocalVal=new ThreadLocal<Integer>(){
        @Override
        protected Integer initialValue() {
            return 1;
        }
    };
    
    public Integer getThreadLocalVal() {
        return threadLocalVal.get();//fixme 返回的只是Integer变量的副本而已
    }

    @Override
    public void run() {
        Integer i=threadLocalVal.get();
        while(true){
            System.out.println(i++);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new ThreadLocalDemo()).start();
        new Thread(new ThreadLocalDemo()).start();
    }
}
```


