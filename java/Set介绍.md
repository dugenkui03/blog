#### 一.基本介绍
相比于List和Map的其他容器，Set最大的特点是不能存放相同的元素，或者是相同而且hashCode相同的元素。三者之间关系如下：
![](https://wx1.sinaimg.cn/mw690/006Xp67Kly1fnn5yaxw0dj30dk0a5mxh.jpg)

##### 1.1特点
Set有HashSet、LinkedHashSet和TreeSet等，最常用的是HashSet，因为速度最快。同Map一样，
1. Set：放入Set中的元素必须重载父类equals方法以保证其唯一性，相同则<font color=red>**不能放进去**</font>；
2. HashSet：优先考虑使用，速度快，必须重载hashCode()。**不唯一但是hashCode相同的元素也可以插入同一个set中**；
3. LinkedHashSet保持着插入的顺序，速度比HashSet稍慢，也必须重载hashCode()；
4. TreeSet将元素存储在红黑树中，保值着元素序列的有序性——插入元素必须实现Comparable接口并重载方法以设置比较逻辑。示例如下：
```
class TreeType extends SetType implements Comparable<TreeType>{
    public TreeType(int i) {
        super(i);
    }
    @Override
    public int compareTo(TreeType o) {
        return (o.i<i?-1:(o.i==i?0:1));
    }
}
```
由以上可总结插入元素可能需要实现的三个重要方法：
- `equals(Object obj)`保证元素唯一性(HashSet则不用完全依赖)；
- `int hashCode()`确定元素插入xxxHashSet中的位置；
- 放入TreeSet<T>的元素类T必须实现`Comparable<T>`接口以确定怎样排序;
##### 1.2基本常用方法
- boolean setX.containsAll(setY)，查看set之间是否有包含关系；
- boolean setX.removeAll(setY)。将与setY中相同的元素从setY中全部移除；

#### 二.示例代码
一直有三个类分别实现了equals(SetType)、equals+hashCode(HashCode)、equals+Comparable<T>(TreeType):
```
//实现了equal方法
class SetType{
    int i;
    public SetType(int i) {
        this.i = i;
    }
    @Override
    public boolean equals(Object obj) {
        return obj instanceof SetType&&(i==((SetType)obj).i);
    }

    @Override
    public String toString() {
        return Integer.toString(i);
    }
}

//实现了equal()和hashCode()，不可重复犯官
class HashType extends SetType{
    public HashType(int i) {
        super(i);
    }

    @Override
    public int hashCode() {
        return i-5;
    }
}

//实现了equal和Comparable，可以重复放入
class TreeType extends SetType implements Comparable<TreeType>{
    public TreeType(int i) {
        super(i);
    }
    @Override
    public int compareTo(TreeType o) {
        return (o.i<i?-1:(o.i==i?0:1));
    }
}
```
驱动类中的test方法将分别将三个类的元素集合放进一个Set中三次；
```
    /**
     * 在Set中放入10个以递增数字为构造函数参数的指定类；
     * @return
     */
    static <T> Set<T> fill(Set<T> set,Class<T> type){
        try{
            for (int i = 0; i < 10; i++) {
                set.add(type.getConstructor(int.class).newInstance(i));
            }
        }catch(Exception e){
            throw new RuntimeException(e);
        }
        return set;
    }

    /**
     * 声明构造参数不同的type类型的元素10个，放进Set中三次
     */
    static <T> void test(Set<T> set,Class<T> type){
        fill(set,type);
        fill(set,type);
        fill(set,type);
        System.out.println(set);
    }
```
实验结果如下：
```
        //只插入10个元素:因为插入元素equals()且hashCode相同
        test(new HashSet<HashType>(),HashType.class);
        //10个元素，因为equals；保持元素插入顺序
        test(new LinkedHashSet<HashType>(),HashType.class);//输出保持插入的顺序
        //以在元素中自定义的大小逻辑排序
        test(new TreeSet<TreeType>(),TreeType.class);
        
        
        //30个元素全部插入了：因为插入元素没有实现hashCode所以可以认为值不同，放进了不一样的位置
        test(new HashSet<SetType>(),SetType.class);
        test(new LinkedHashSet<TreeType>(),TreeType.class);

```


