#### 参考
1. [Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html);

2. 源码

##### 注意事项

1. key允许为null，同一个key会更新之前的value；
2. 扩容`resize()`分两种情况：元素数量大于容量与装载因子的积，或者**某个桶中元素数量超过8，若容量(即桶的数量)小于64，即对其扩容;
3. 放入`HashMap`中的桶中的红黑树的大小关系通过hashCode确定。**TreeMap中没有hash实现，其大小关系是通过在构造参数中指定`Comparator<? super K> comparator()`实现。

#### 一. 2的幂

##### 1. 为什么HashMap容量是2的幂：

因为最常用的操作不管是`put()`还是`get()`都是使用`key`的`hashCode()`与HashMap容量减1按位与操作来进行取模，而HashMap容量减1的二进制表示是一连串`1111`——假设一共有n位，则以上使用按位与取模的结果其实是key的`hashCode`低n位。

假设HashMap的容量不是2的幂，容量减1的二进制表示方式肯定有某些位不为1，,而`hashCode`与其对应位按位与操作的结果总是0，所以`hashCode`的这个位置为0或者为1存放的位置都是为0时的位置。

简而言之：
1. 是为了使用按位与代替%进行取模更快；
2. 支持取模的值尽可能均匀的分布。


##### 2. 2的幂的相关操作

###### 2.1 找到大于或等于给定int型数的最小的2的幂

```
    static final int tableSizeFor(int cap) {
        int n = cap - 1;//fixme 如果不执行-1操作，则如果cap原本就是2的幂，则返回值是CPA*2
        n |= n >>> 1;//n和n>>>1两个，原最高位的1和低一位的位置都是1，则或操作后原来最高位的1和低1位的位置都是1
        n |= n >>> 2;//此时n最高两位和n>>>2最高两位低两位的位置都是1，则或操作后原来最高位的1和低3位的位置都是1
        n |= n >>> 4;//。。。原来最高位和低7位都是1
        n |= n >>> 8;//原来最高位和低15位都是1，
        n |= n >>> 16;//考虑极端情况高位1在31位，按位或进行到此步也可以覆盖到所谓低位了

        //返回值：如果是负数则返回2的0幂，即1，如果计算结果大于阈值，则返回阈值，否则返回n+1，是一个2的幂
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

1. 刚刚大于等于一个数的2的幂或者是这个数本身——**此时这个数的二进制表示是仅仅有一个1**；

2. 或者(是二进制表示)在比最高位的1的位置加一个1，其他位置置为0；

3. 可以通过将这个数减1将两种情况置为第二种情况，然后使用以下按位或的方式计算：因为是第一种情况的话按照上述方法进行计算后求得的是自身，如果是第二种情况的话减1高位也不会变。<font color=red>**进行计算最重要的信息就是最高位1的位置**</font>
4. 计算的方式是不断将现高位的1覆盖到临近低位，起始一个1所以只能覆盖以为，以2的幂递增；
5. 返回值时要判定：
    - 如果n为负数表示之前小于等于0，则结果为1，即2的0次幂；
    - 如果结果大于阈值`2^30`，则将其设置为阈值即可——注意容量是int类型而且需是2的幂，int类型最大值是`2^31-1`，所以符合条件的最大值即阈值时`2^30`;

<font color=red>**最方便的理解方式是背会这段代码**</font>

###### 2.2 计算key的hashCode

```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashMap的key的hashCode计算方式是调用父类方法hashCode()结果与其高16位**按位异或**，不直接采用hashCode的原因是：

1. 表的容量是2的幂，如果在计算索引位置时，hashCode仅仅是高位发生变化那么计算结果即索引位置总会发生碰撞，而使用位运算取出高16位，与hashCode进行按位异或是系统资源消耗最小的、而且能够考虑到hashCode高位影响的方式；
2. 使用异或是因为按位与和按位或每位计算结果不均匀，即前者出现0的概率比较高，后者出现1的概率比较高。


###### 2.3 找节点位置代码

对节点各种需要查找的操作都有如下代码，就是**hashCode与1字串按位与的方式**求索引的方法：

```
((e = tab[index = (n - 1) & hash]) != null)

(p = tab[index = (n - 1) & hash]) != null) 

(first = tab[(n - 1) & hash]) != null)

((p = tab[i = (n - 1) & hash]) == null)

```


#### 二.HashMap相关操作：增、查、扩容

##### 1.HashMap中的信息节点

HashMap有两种主要节点，分别是链表节点`Node`和树节点`TreeNode`，后者是前者的孙子类，而`Node`继承自`Map.Entry`。试着之间的继承关系如图所示：

![](https://wx1.sinaimg.cn/mw1024/006Xp67Kly1fqvr4yydftj30or0g7my9.jpg)

###### 1.1 Node节点
在`HashMap`中，`Node`节点存放在数组中作为哈希桶数组，`transient Node<K,V>[] table`。节点主要信息有：hash值、key值、value值和next节点；节点上定义的操作有：以以四个值为参数的构造器、为节点设置value值、做相等比较和计算哈希值：

```
static class Node<K,V> implements Map.Entry<K,V> {
        //fixme: 四个实例变量
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        //构造参数
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        //返回此节点的key或value
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        //打印节点
        public final String toString() { return key + "=" + value;}

        //fixme: 哈希值是key和value哈希值的按位异或；
        //注意其在哈系统中存放位置是实例变量hash与哈希桶Node<K,V>[]长度按位与求得
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        //为此节点设置新value——没有设置key的方法，
        //因为key决定了hash值，hash值决定了存放位置，修改的话需要改很多
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        //计算两个节点是否相等：1. 是否是同一个节点；
        //2. key和value用equals方法判定是否同时为true；
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

###### 1.2 TreeNode节点

如图`TreeNode`继承自`HashMap.Entry`，除了Node中的四个属性`hash/key/value/next`和`HashMap.Entry`中的两个节点`before/after`外还有表示红黑树节点的五个属性`parent、left、right、prev(删除节点是用到)、red`。



