#### 一.基本事实

##### 1. 线程安全和效率

1. `HashMap`：线程不安全；
2. `HashTable`：线程安全；
3. `Concurrent`：线程安全，锁力度比`HashTable`细，迭代方式是弱一致性。