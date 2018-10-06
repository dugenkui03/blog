
参考博文：
[why-not-to-start-a-thread-in-the-constructor-how-to-terminate](https://stackoverflow.com/questions/5623285/why-not-to-start-a-thread-in-the-constructor-how-to-terminate)

《java并发编程实战》中讲到不要在构造器中启动线程或者调用可改写的实例方法，因为会导致`this`逸出，从而发布一个尚未完成的对象。

###### 示例
- 构造参数中启动线程
```
public class EscapeDemo1 implements Runnable{
    private int a;
    private String str;

    public EscapeDemo1() throws InterruptedException {
        new Thread(this).start();//fixme 尚未构造完成的对象被发布
        TimeUnit.MILLISECONDS.sleep(11);
        str="aa";
    }
    
    @Override
    public void run() {
        a=str.length();//抛出异常
    }
    
    public static void main(String[] args) throws InterruptedException {
        new EscapeDemo1();
    }
}
```
- 调用可被改写的方法
```
class EscaFath{
    int a;
    int b;

    public EscaFath() {
        this.a = 1;
        func();
        this.b = 1;
    }

    public void func(){ }
}

public class EscapeDemo1 extends EscaFath{
    public EscapeDemo1() {
    }

    //改写父类的方法，输出未构造完成的父类对象
    @Override
    public void func() {
        System.out.println(a+":"+b);
    }

    public static void main(String[] args) throws InterruptedException {
        new EscapeDemo1().func();
    }
}

```

