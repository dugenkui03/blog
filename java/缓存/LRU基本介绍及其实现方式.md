**参考**
- [LRU算法](http://flychao88.iteye.com/blog/1977653)
- [dubbo-cache](https://github.com/dugenkui03/incubator-dubbo/tree/3.x-dev-cache/dubbo-filter/dubbo-filter-cache/src/main/java/org/apache/dubbo/cache)

#### 一.基本介绍

##### 1.1 常见缓存淘汰算法及其实现思路

对于缓存，常见淘汰算法有3：
1. `FIFO`: first in first out,先进先出，即**假定刚刚加入的数据总会被访问到**；
2. `LRU`：least recently used，最近最少使用，判断最近被使用的时间，**假定未被使用的时间越久就不可能在被使用**；
3. `LFU`:least frequently used，**数据使用次数最少的，优先被淘汰**。


对于FIFO算法，即`caffeine`中的`expireAfterWrite`方法，**仅仅在数据插入时FIFO即可，LRU算法则在调用get()方法时再将数据重新插入即可**。`LFU`则将数据根据调用次数对数据进行**排序**。


##### 1.2 `LRU`不同版本介绍

`LRU`分为 `LRU`、`LRU-K(2)`、`two queues（2q)`和`multi queue(MQ)`四个版本。命中率、代价和复杂度对比如下：


对比项目 | 排序
---|---
命中率| LRU-2 > MQ > 2Q > LRU
复杂度 | 同上
时空复杂度/代价 |  同上

###### 1）LRU

LRU least recently used 最近最少使用。**根据数据的访问历史淘汰数据，实现是建立在链表上的。新插入和被访问的节点放在表头，删除时从链尾开始**。

![5b7bb8a9-43bd-4248-a831-f4f2358de29b.png](WEBRESOURCE2a63bfe7640e80e6ee680bf7d9d7fc36)

least recently used 最近最少使用在存在热点数据时命中率高。但是缺点如下：
1. 锁力度比较粗，比如`ConcurrentHashMap`的分段锁写入性能好；
2. 由于LRU给新加入的节点放进了链表头部—假定其拥有很大的quan zhi权值，因此如果出现**周期性数据并且数据大小接近缓存时，则会污染缓存—将有效缓存数据全部挤出缓存**。


###### 2）LRU-K

相比LRU，LRU-K需要维护一个**访问历史队列**，用于记录所有缓存数据被反问历史，只有数据被反问次数达到k时，才将数据放进**缓存数据队列**。

![aaa.png](WEBRESOURCE6de89395db144a54a81c4f5d93713d1b)

1. 数据第一次被访问时，加入“访问历史表”；
2. 数据在“访问历史表”中按照FIFO或者LRU算法淘汰；
3. 数据在“访问历史表”中被访问k次后：**将数据引用从“访问历史表”移至“缓存队列表”中，并且按照时间排序—就是新加入或者被访问的数据放在表头**；
4. 综合考量LRU-2为最优选择，**虽然跟大的K值会让命中率更高，但是适应性差，需要大量访问才能将数据从“历史队列”清除**。


优缺点：
- 相比**超时缓存**，LRU的链表结构决定其写入性能受阻；
- 因为要维护没有放入缓存的对象，因此比LRU占用更多的内存;

###### 3) two queue

同样是两个队列-**第一个队列使用FIFO算法**。

![444.png](WEBRESOURCE3c4f7624c42e79561e0bfd1d4e6da686)

1. 新访问的数据放进FIFO队列；
2. FIFO队列数据没有在被访问则被淘汰；有则将数据插入到LRU队列头部，再次被访问是则再将数据移动到队列头部；
3. LRU队列淘汰末尾的数据。

性能：
2Q性能和命中率同LRU-2，但是 **<font color=red> 2Q会减少一次从原始存储读取数据或者计算数据的操作**</font>。

###### 4）Multi-Queue
MQ算法根据数据**访问频次**将数据放进**访问优先级**不同的多个队列，访问时优先访问优先级比较高的队列。如图：
![444.png](WEBRESOURCE98dbbf893a35c73abcf55b81b71940ff)

注意点：
- 新插入的数据放到优先级最低的Q0，每个队列按照LRU算法管理数据；
- 当数据访问次达到一定次数时，将数据移至更高优先级队列。**<font color=red> 当数据在指定时间未被访问时，需要降低优先级**<font>;
- Q-history记录了从缓存中淘汰的数据，并且记录**数据引用和被调用次数，<font color=red>如果数据在Q-history中被重新访问，则计算其优先级，移至相应队列头部；**
- 由数据降低优先级策略可知，MQ需要记录并定时扫描数据的最近被访问时间，因此代价比LRU高。

#### 二.实现方式

##### 2.1 组合方式`linkedHashMap`

因为java的**单根继承**，因此**组合应该优于继承，便于扩展**。

```java
public class LruCache<K,V> implements Cache<K,V> {
    private final Map<K,V> cache;

    public LruCache(final int maxSize){
        cache=new LinkedHashMap<K,V>(maxSize){
            //返回值表示是否进行移除操作，此方法在节点插入操作中被调用，如putVal、compute、computeIfAbsent、merge
            @Override
            protected boolean removeEldestEntry(Map.Entry eldest) {
                return this.size()>maxSize;
            }
        };
    }

    @Override
    public V get(K key) {
        return cache.get(key);
    }

    @Override
    public void put(K key, V value) {
        cache.put(key,value);
    }

    @Override
    public boolean remove(K key) {
        //remove返回之前与key绑定的value—如果不为null，表示移除的key对应的valu不为null；
        // 为null，返回false表示没有与key对应的value
        return cache.remove(key)!=null;
    }
}
```




##### 2.2 继承`LinkedHashMap`

重载`removeEldestEntry`方法即可。想要线程安全则使用`synchronized`修饰方法即可。
```java
import java.util.LinkedHashMap;
import java.util.Map.Entry;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private static final long serialVersionUID = 1L;
    protected int maxElements;

    public LRUCache(final int maxSize) {
        super(maxSize, 0.75F, true);
        this.maxElements = maxSize;
    }

    /*
     * 返回值表示是否进行数据移除
     */
    @Override
    protected boolean removeEldestEntry(Entry<K, V> eldest) {
        return (size() > this.maxElements);
    }
}
```
