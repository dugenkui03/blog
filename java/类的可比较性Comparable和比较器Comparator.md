##### 1. `Comparable`

`Comparable`源码如下：

```
public interface Comparable<T> {
    public int compareTo(T o);
}
```

类实现该接口意味着该类是可比较的，其方法返回值`-1,0,1`分别表示当前类比进行比较的类(方法参数)小、相等、大。

**实现该接口意味着该类有了‘可比较’的属性**。

##### 2. `Comparator`

`Comparator`部分源码如下：

```
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);

...

    default Comparator<T> reversed() {
        return Collections.reverseOrder(this);
    }
    
...
   public static <T extends Comparable<? super T>> Comparator<T> reverseOrder() {
        return Collections.reverseOrder();
   }
...

```

如果对没有实现`Comparable`接口的类进行排序，那么可以使用比较器`Comparator`实现。他可以选出类中一些“信息”进行比较，比如变量甚至哈希码。

如源码所示该接口是函数式接口，其唯一抽象方法是`int compare(T o1,T o2)`，其返回值`-1,0,1`分别表示第一个参数比第二个参数小、相等、大。


**<font color=red>如果对一个实现`Comparator`的类指定比较器进行比较的时候，比较规则与比较器对齐</font>**。示例如下：

```
class DemoClass implements Comparable{
    public int a;

    public DemoClass(int a) {
        this.a = a;
    }

    @Override
    public int compareTo(Object o) {
        return (a<((DemoClass)o).a)?-1:(a==((DemoClass)o).a?0:1);
    }

    @Override
    public String toString() {
        return a+"";
    }
}

public class Test2 {
    public static void main(String[] args) {
        List<DemoClass> list=new ArrayList();
        list.add(new DemoClass(3));
        list.add(new DemoClass(1));
        list.add(new DemoClass(2));

        System.out.println(list);

        //用元素自身比较属性进行排序
        Collections.sort(list);
        System.out.println(list);

        //用比较器进行排序

        Collections.sort(list,(x,y)->{//此时x和y都是list中元素类型
            return x.a<y.a?1:(x.a==y.a?0:-1);//逻辑与类定义的比较方式相反
        });

        System.out.println(list);
    }
}

output:
[3, 1, 2]
[1, 2, 3]
[3, 2, 1]


```


