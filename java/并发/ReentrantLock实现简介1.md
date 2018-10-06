
![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fr7r5iopewj30pa0a3q39.jpg)

如上图所示，除了三个有公平和非公平属性的锁分别是`ReentrantLock、ReentrantReadWriteLock和Semaphore信号量`。

#### 一.基本变量

##### 1.1 抽象类`Sync extends AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer`及其实现类`FairSync公平锁`和`NonfairSync`

###### 1.1.1 `Sync`

`AbstractQueuedSynchronizer 和 AbstractOwnableSynchronizer`虽然是抽象类，但是并没有抽象方法。

在`ReentrantLock`中，`Sync`是公平锁和非公平锁的抽象父类，并有**抽象方法`abstract void lock();`提供给子类实现两种方式的加锁。其他多个方法则都用`final`修饰，不能被子类修改。

```



```



