[博文.基本介绍](http://ifeve.com/java-blocking-queue/)

##### 基本介绍
阻塞队列即支持阻塞插入和阻塞出队的队列——一直等到队列有空位或者有元素。

阻塞队列每种方法提供了四种操作；
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fpmi78mhnhj30rv08faan.jpg)

##### 1.BlocingQueue基本方法和重点关注
- `put()、offer()和add()`都会想队列中添加元素，不同的是当队列满了以后，`put()`会发生阻塞等待一段时间，以加入元素，`offer()`会直接返回==FALSE==，`add()`会抛出异常；
- `take()、poll()和remove()`都是删除队头，队列为空时各自的表现是：`take()`会发生阻塞直到队列有元素，`poll()`返回null，`remove`则会抛出异常；

综上，我们需要关注`put()`和`take()`方法在`BolcingQueue`中的实现，他们的实现和Lock对象紧密相关；

`BolcingQueue`接口在java中有两个主要实现:`LinkedBlockingQueue`和`ArrayBlocingQueue`,前者是线程池工作队列默认的使用，因为工作队列经常需要增删。`PriorityBlockingQueue`是另一个重要实现；

```
graph BT
A[ArrayBlocingQueue]-->B[BolcingQueue]
LinkedBlockingQueue-->B
PriorityBlockingQueue-->B
```

##### 2.`LinkedBlockingQueue`数据变量及`put()`和`take()`

###### 2.1数据节点

`LinkedBlockingQueue`同平时使用的链表一样，数据放在节点中(静态内部类)，节点中定义了数据变量和下一个节点，具体实现代码如下:
```
static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }
```
###### 2.2最大容量、当前大小、队头、队尾
```
    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * 队头不存数据
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * 队尾的next是null
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;
```

###### 2.3 加锁对象：存取元素说和相应Condition

队头取元素锁和相应Condition-notEmpty,：队尾放元素锁和相应Condition-notFull。
```
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```
###### 2.4 `take()`
获取队头元素，队列为空则等待
```
   public E take() throws InterruptedException {
        E x;//返回结果
        int c = -1;
        final AtomicInteger count = this.count;//当前大小
        final ReentrantLock takeLock = this.takeLock;//读数据锁
        takeLock.lockInterruptibly();//阻塞获取锁
        try {
            while (count.get() == 0) {//当队列为空时，线程挂起
                notEmpty.await();
            }
            x = dequeue();//获取队头元素
            c = count.getAndDecrement();//返回之前大小，并将当前大小递减1
            if (c > 1)//如果之前队列元素大于等于2，则唤醒其他获取队头元素的线程；
                notEmpty.signal();
        } finally {
            takeLock.unlock();//释放锁
        }
        if (c == capacity)//如果之前大小等于容量，则唤醒其他等待添加元素的线程向队列添加元素；
            signalNotFull();
        return x;
    }

... 出队函数，被三个元素获取函数调用，而且这三个函数加的是同一个锁。

    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }

... 发信号给所有挂起的等待添加元素的任务
    /**
     * Signals a waiting put. Called only from take/poll.
     */
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
```
###### 2.5`void put(E)`

```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();//如果插入元素为null则抛异常

        int c = -1;//当前大小
        Node<E> node = new Node<E>(e);//需要入队的节点
        final ReentrantLock putLock = this.putLock;//插入锁
        final AtomicInteger count = this.count;//当前大小
        putLock.lockInterruptibly();//尝试加锁
        try {
            while (count.get() == capacity) {//如果队列已满
                notFull.await();//挂起
            }
            enqueue(node);//入队
            c = count.getAndIncrement();//返回入队前大小并更改当前大小
            if (c + 1 < capacity)//如果当前大小小于队列定义容量，唤醒所有插入进程
                notFull.signal();
        } finally {
            putLock.unlock();//释放锁
        }
        if (c == 0)//如果之前队列为空，即现在不为空，则唤醒所有等待获取元素的任务
            signalNotEmpty();
    }
    
...入队代码
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
...
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```

##### 补充：`PriorityBlockingQueue`的使用方式
