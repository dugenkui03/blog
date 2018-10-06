#### 前言
本文主要介绍TreeMap、HashMap、LinkedHashMap和ConcurrentHashMap。他们之间的大致关系如下图所示：
![image](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fnn3b30aekj30l209oweq.jpg)

这四种Map各自的特点如下：
- HashMap作为编程的首选项，速度最快；
- LinkedHashMap 取“键值对”的顺序是其插入的顺序，速度比HashMap慢一点，但是遍历迭代的速度更快；
- TreeMap 基于红黑树的实现，所得到的结果可以经过自定义的排序类进行排序，含有获取子树的方法；
- ConcurrentHashMap 线程安全的Map；

Map中键必须是唯一的，值可以重复，<font color=red>**如果键值与或者是根据自定义的“比较类”在逻辑上是相同大小，则会覆盖前一个值**</font>

##### 1.SortedMap/TreeMap
SortedMap是基于红黑树的实现，如右图所示，TreeMap使其唯一的实现类，使用SortedMap可以确保键处于排序的状态，其特性如下：
- `Comparator`类作为构造器参数，重载其compare(obj1,obj2)方法自定义排序方式；
- `T firstKey()和 T lastKey()`返回当前有序的Map第一个和最后一个键；
- `SortedMap subMap(fromKey,toKey);SortedMap headMap(toKey); SortedMap tailMap(fromKey)`生成Map的子集/子树，分别指定了起始key值、指定起始值、指定结束值；

重申：<font color=red>**如果键值与或者是根据自定义的“比较类”在逻辑上是相同大小，则会覆盖前一个值**</font>
```
public class TreeMapDemo {
    public static void main(String[] args) {
        TreeMap<Integer,String> linkedMap=new TreeMap<>(
                new Comparator<Integer>() {//与key类型对齐
                    @Override//根据参数o1与参数o2比较大小返回正、0、负，小的在前边；
                    public int compare(Integer o1, Integer o2) {
                        return Math.abs(o1-2)-Math.abs(o2-2);
                    }
                }
        );
        linkedMap.put(1,"du");
        linkedMap.put(2,"gen");
        linkedMap.put(3,"kui");//因为根据自定义的比较类，3和1在逻辑上是大小的（距离2）,因此会覆盖（1，“du");
        linkedMap.put(1,"gen");
        System.out.println(linkedMap);//输出顺序为从小到大
    }
}
```


##### 2.LinkedHashMap
LinkedHashMap有如下特点：
- 速度比Hash稍微慢，但是遍历迭代的速度比HashMap块；
- 以元素插入的顺序打印元素，但是
- 可以在构造器中设置是否采用基于访问的“最近最少使用算法LRU”，即get(key)最少的元素使用默认方式打印时放在最前边；

示例代码如下：

```java
public class LinkedHashMapDemo {
    public static void main(String[] args) {
        //沟槽参数分别是初始化容量、装载因子和是否开启LRU
        LinkedHashMap<Integer,String> linkedMap=new LinkedHashMap<>(10,0.75f,true);
        linkedMap.put(1,"du");
        linkedMap.put(2,"gen");
        linkedMap.put(3,"kui");
        System.out.println(linkedMap);//以装载方式打印

        linkedMap.get(2);
        System.out.println(linkedMap);//可以看到被访问的元素靠后打印了
    }
}
```





