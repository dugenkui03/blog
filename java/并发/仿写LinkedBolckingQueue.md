##### 1.引言

LinkedBolckingQueue以链表的方式实现了一个线程安全的队列。主要使用的技术如下：

1. `ReentrantLock`(重入锁)和`Condition`；
2. 原子数据类型`AtomicInteger`;
3. 函数接口`Consumer`;

实现此简单的队列后，通过**依次去掉某个技术模块，我们看看多个线程同时存取队列时的效果。**



##### 2.简单实现

###### 2.1数据结构

首先是实现队列的主要代码：泛型节点、final容量、当前大小(final 原子类)、头引用、尾引用(transient)等五个部分。

```
    //在构造函数中初始化后不再改变
    private final int capacity;
    //当前大小，引用不变，但是其值可变
    private final AtomicInteger count=new AtomicInteger();
    //队头队尾
    transient Node<E> head;
    transient Node<E> last;
    //参数为节点值的构造参数、getter和setter方法
    static class Node<E>{
        E val;
        Node next;

        public Node(E val) {
            this.val = val;
        }
    }
    
    //初始化队列时：1.检查容量值；2.初始化队头队尾（当前大小类的初始化时已经是0）；
    public LinkedBlockingQueueA(int capacity) {
        this.capacity = capacity;
        
        this.capacity = capacity;
        last=head=new Node<E>(null);
    }

```

###### 2.2 线程安全保证`ReentrantLock`和`Condition`

因为要检查从队列中取元素，并需要**唤醒在队列为空时等待的线程**。因此需要一下类(入队同理，也需要唤醒**队列不满时等待将元素入队的线程**):
```
    private final ReentrantLock takeLock=new ReentrantLock();
    //fixme:唤醒在队列为空时做等待状态的取元素线程
    private final Condition notEmpty= takeLock.newCondition();

    private final ReentrantLock putLock=new ReentrantLock();
    //唤醒队列已满时，等待插入元素的线程
    private final Condition notFull=putLock.newCondition();

```

###### 2.3 入队操作

入队操作注意几点：

1. `Condition`类的`await()、signal()`方法的调用都是获取锁后进行的；
2. 入队步骤是：
    1. 阻塞获取可中断锁；
    2. 检查是否有空间，无则wait，占用线程池一个线程；
    3. 入队并调整尾引用；
    4. 查看是否还有容量，有的话则唤醒其他等待putLock锁的入队线程；
    5. 释放锁；
    6. 检查入队之前队列是否为空，为空的话则唤醒可能等待出队的线程；

```
    public void put(E e) throws InterruptedException {
        //检查插入节点值不应为null
        if(e==null) throw new NullPointerException();

        Node<E> node=new Node<>(e);
        int c=-1;
        //阻塞知道获取锁，同时接受中断信号
        putLock.lockInterruptibly();
        try{
            //队列已满则等待,while,putLock.newCondition()对象
            while(count.get()==capacity){
                notFull.await();//放弃锁
            }
            //获取锁后：入队，更新当前大小，并返回之前大小：
            //fixme 这种操作顺序也防止了其他想要加锁的线程获取到
            //声明一般的锁
            last=last.next=node;
            c=count.getAndIncrement();//
            //如果当前大小小于容量则有空位，唤醒其他入队线程；
            if(c+1<capacity){
                notFull.signal();
            }
        }finally {
            putLock.unlock();//释放锁
        }
        //在加入这个节点之前队列为空，则可能有线程在等待出队，则调用出队锁对应的Condition对象唤醒这些线程
        if(c==0){
            //fixme 注意一定要先获取锁，才能调用awati()和notify()方法
            takeLock.lockInterruptibly();
            notEmpty.signal();
            takeLock.unlock();
        }
    }
```

- 出队 todo 更加复杂
















##### 3.其他

链表：插入删除快，查找慢；

队列：先进先出；

线程安全：多个线程访问某个类，主调代码不需要额外的同步，这个类就能表现出正确的行为，我们就成这个类是线程安全的。比如LinkedBlockingQueue使用Lock加锁保证任何；



**[补充]：对象序列化与transient(adj:短暂的)关键字:**

序列化的类需要继承Serializable接口(或者其父类继承)，其中不想被序列化的变量用transient实现。注意：

1. transient只能修饰变量；
2. 静态变量不会被序列化，不管是否有transient；
3. 如果实现对象序列化用的是`Externalizable`，则transient没什么用了；
4. 发送端和接收端都需要有一个版本号`serialVersionUID `;
5. [补充及示例代码](http://note.youdao.com/noteshare?id=90d4c1d79432c90746451a53a7d73afa&sub=B9FC0BCECFB04BF0988FD723F382A75B)


**`Lock`获取锁的三个方法：**
##### 4.`lock()、tryLock()和tryInterruptibly()`的区别
- `void lock()`：调用后一直阻塞知道获取锁，对应==无限制锁==；
- `boolean tryLock()`：尝试获取锁，不能则返回FALSE，对应==定时锁==，通过while可作为==轮询锁==；
- `void lockInterruptibly()`：同`lock()`，但是接受中断信号，对应==可中断锁==；
