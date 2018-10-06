##### 参考博文
- [深入理解Java内存模型（四）——volatile](http://www.infoq.com/cn/articles/java-memory-model-4#)
- []
- 

##### 1.引言

`volatile`变量具有以下特性：
1. 可见性：每次读取volatile变量，总是从主存刷新值；每次更新volatile，总是立即同步到主存；
2. 原子性：对于任意<font color=red>**单个`volatile`变量的读写具有原子性。但是被写入volatile变量的值必须独立于任何程序状态，诸如自增就不是原子的。**</font>


###### `volatile`读写内存语义：
- 写的内存语义：当某个线程写一个`volatile`变量时，JMM会把该线程工作内存**所有共享变量刷新到主内存——不仅仅是某个`volatile`**。换句话说：线程a写某个volatile变量而线程b随后读取**同一个**volatile变量，那么线程a在写volatile变量之前对其可见的所有变量，对线程b读取此volatile变量后也是可见的;
- 当某线程读取一个`volatile`变量时，JMM会把该线程对应的工作内存**所有共享变量**设置为无效，然后从共享内存读取共享变量；

如上，volatile保证的是整个工作内存所有共享变量的可见性，而不仅仅是volatile变量：

通过`volatile`实现线程间的通信：
1. 线程a写一个volatile变量，实质上是向接下来要读取这个volatile变量的某个线程发送消息；
2. 线程b读取volatile变量，实质上是接受之前某个写volatile变量的线程发送的消息；
3. 线程a写volatile变量和**随后**线程读取这个volatile变量，实质上就是两个线程通过主内存发送接收消息。

##### 2. 使用场景

1. 当多个线程读取volatile变量，一个线程写时，可以使用；
2. 当多个线程读取volatile变量，多个线程写，而且对**<font color=red>`volatile`变量的写操作不依赖任何程序状态值时，即不依赖当前值，也未包含在其他变量的不变式中(low<up)</font>**，可以使用;
3. 但是如果多个线程写，而且依赖程序状态值，比如自增操作依赖volatile当前值，则不能用volatile变量。

注意volatile不提供必须的原子性，**限制指令重排最大的作用还是对其可见性的影响，<font color=red>限制指令重排的内存屏障作用于单个线程的工作内存中</font>**。


##### 3.使用模式

###### 3.1状态标识

volatile变量是一个布尔类型的状态标识，指示发生了一个重要的一次事件：
```java

volatile boolean shutdownRequested;

...

public void shutdown(){ shutdownRequested=true;}

public void doWork(){
    while(!shutdownRequested){
        //doSomething
    }
}

```
###### 3.2 一次性安全发布 one-time safe publication

类似DCL单例模式，或者如下代码：
```

class SomeOtherClass{
    BackgroundFloobleLoader floobleLoader=new BackgroundFloobleLoader();

    public void doWork(){
        while(true){
            //todo doSomething
            if(floobleLoader.theFlooble!=null){
                //todo doSomething about theFlooble;
            }
        }
    }
}

public class BackgroundFloobleLoader {
    public volatile Flooble theFlooble;

    public void initInBackground(){
        //todo doSomething
        theFlooble=new Flooble();//fixme 唯一写操作
    }
}
```

###### 3.3 将volatile变量作用于多个观察结果的发布

即一个线程写操作或者多个线程写操作但是不依赖于程序状态，多个线程读取的场景。

###### 3.4 开销较低的“读写锁”

例如自增计数器，读操作通过volatile保证可见性，写操作通过加锁同步；

```java
public class CheesyCounter{
    private volatile int value;
    public int getValue(){ return value; }
    
    public synchronized int increment(){
        return value++;
    }
}
```


