

##### TODO

- 容器数组使用volatile变量：保证修改后能够是的所有后续的读(写)线程能够立即感知到修改，但是为什么不进行实际写入时仍然要通过相关操作保证“volatile写语义”，见`set(E v,int index)`代码相关操作；

![](https://wx4.sinaimg.cn/mw690/006Xp67Kly1fr1pqzbd2vj30c907et98.jpg)

如图所示：写`volatile`屏障指令插入策略可以保证在`volatile`写之前，所有写操作都已经刷新到主存对所有处理器可见了。其后全能型屏障指令为了避免写`volatile`与其后`volatile`读写指令重排序。

- 原子性：对任意**单个volatile变量的读/写具有原子性**，但类似于volatile++这种**复合操作不具有原子性**；


##### 1.基本思想

`copy-on-write`适用于<font color=red>**迭代操作远远大于修改操作,比如事件通知系统**</font>的情况。

“写入时复制”容器发布一个**事实不可变对象**，这样访问时就不需要进一步同步。而**修改是通过创建并重新发布底层数组实现。通过读写分离，正在迭代iterate的操作是通过引用指向的旧数组**：
```
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}

/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}
```

 - 由于事实不可变，因此对其同步只需保证数组内容的可见性；
 - 多个线程同时对数组进行迭代或者修改操作，不会相互造成干扰。写入时复制容器不会抛ConcurrentModificationException；
 - 返回的元素与创建时的元素完全一致。


##### 2.遍历操作

遍历操作通过返回包含list的快照`class COWIterator<E> implements ListIterator<E> extends Iterator<E>`。所谓快照就是指向事实不可变的容器Object数组的引用。

通过`COWIterator`相关接口可以实现向前遍历和向后遍历`hasNext() hasPrevious() next() previous()`。

其基本思想和源码如下：
```java
    /**
     * 返回迭代器，包含链表中元素的快照：
     *      所谓的快照是指向事实不可变的数组的引用，当修改容器的线程为对象赋值新的数组时，
     *      快照(引用)依然指向旧的数组对象，由此实现读写分离。
     * 因为迭代的是快照，因此无需使用同步。
     */
    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }
```
    
```java
    //ListIterator extends Iterator
    static final class COWIterator<E> implements ListIterator<E> {
        /** 数组的快照*/
        private final Object[] snapshot;
        /** 指向数组下标的游标 */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            snapshot = elements;
            cursor = initialCursor;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }
        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

        /**fixme 添加、设置、移除操作不被支持
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; {@code remove}
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }
        public void set(E e) {
            throw new UnsupportedOperationException();
        }
        public void add(E e) {
            throw new UnsupportedOperationException();
        }

        //Consumer中唯一抽象方法void accept(T t)：一个参数，无返回值；
        //遍历数组所有的元素，并且以 fixme 从cursor开始的每个元素为参数进行操作，参数引用，不会修改值
        @Override
        public void forEachRemaining(Consumer<? super E> action){
            Objects.requireNonNull(action);
            Object[] elements=snapshot;
            final int size=elements.length;
            for (int i = cursor; i < size; i++) {
                @SuppressWarnings("unchecked")
                E e=(E)elements[i];
                action.accept(e);
            }
            cursor=size;//游标放到链表尾部
        }
    }
```

##### 3. 修改

如第一小节所示，更新(set和add)是通过在拷贝的数组中做修改，然后将元素数组对象指向新数组实现更新的。示例`set`和`add`。

<font color=red>**为防止更新操作在相同备份上做不同跟新，更新需要串行执行，因此都需要获取全局锁**</font>:
```java
-   //替换指定位置的元素；
    public E set(int index,E element){
        final ReentrantLock lock=this.lock;
        lock.lock();//不可中断
        try {
            Object[] elements=getArray();
            E oldValue=get(elements,index);//获取旧值，超出边界则抛异常

            //如果新值旧值不相等
            if(oldValue!=element){
                int len=elements.length;
                /**
                 * fixme 修改数组时:
                 *      1.先拷贝数组;
                 *      2.修改拷贝后的数组；
                 *      3. 为容器底层数组赋新拷贝的值
                 */
                Object[] newElements=Arrays.copyOf(elements,len);
                newElements[index]=element;
                setArray(newElements);
            }else{
                //TODO 并非完全无效的操作，保证volatile变量的写语义：
                // fixme 写语义就是立即将修改过的值更新到主存中去啊，这句有必有？
                //fixme 应该是指令重排：防止volatile写之后的读操作重排序到写的部分
                setArray(elements);
            }
            return oldValue;//fixme 如果有catch而且catch代码块没有抛异常，则return之前可能会有异常被捕获并且无法返回，因此提示缺少返回值。
        }finally{
            lock.unlock();
        }
    }
    
    //添加指定的元素到链表尾部，过程需要加锁，因为可能有多个线程同时加锁，导致拷贝出多个副本，并且在相同位置添加进不同的值
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            //拷贝长度大于数组长度，则用null对齐
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            //为链表尾部元素赋值
            newElements[len] = e;
            //设置容器数组值
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```


