



待更：以下博文缩写均为某一时期所理解，会不断更新。措辞理解多有错误，想要进一步学习请移步其他书籍资料。

#### **一.前言：关于同步**
　　同步执行并非“并行”执行，可以理解为“串行”执行，异步执行可理解为“并行”执行。
　　所谓的“多线程”可以理解为多个工作者，cpu驱动他们去执行各自的工作。因为这些工作者时常会对某一些资源同时产生兴趣，他们都需要知道资源的“现状态”并根据自身逻辑将资源操作为“目标状态”，因此这些工作者对资源进行操作是常常具有排他性，比如自增。
　　synchronized的锁对象类似于一个锁链，任何一个工作者进入这个锁链管理的代码，就可以排他的执行锁链管理的代码。我们写代码的过程相当于给共享资源加锁或者是给操作共享资源的代码加锁的过程。
#### **二.synchronized关键字**
　　synchronized是给资源加锁的方式之一，既可以使用**synchronized方法**或者**synchronized(this)**用当前对象作为锁对象，也可以使用**synchronized(obj)**指定某个对象作为锁对象。即可以给某个方法加锁，也可以给某些代码块加锁。
##### **1.不同锁对象和非synchronized方法**
　　线程a获取某个锁对象下代码的操作权时，线程b可以获取其他锁对象下代码的操作权或者非synchronized代码的操作权。示例代码如下：
　　1）首先我们有资源类，包含操作资源的三种方法：分别加了当前对象锁、新建对象锁以及不加锁：
```
package sync;

import org.omg.PortableServer.THREAD_POLICY_ID;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class Resource {

    synchronized public void func1() throws InterruptedException {
        while(true){
            Thread.sleep(1000);
            System.out.println("func1");
        }
    }

    public void func2() throws InterruptedException {
        synchronized (new Object()){
//      synchronized (this){ 代替上行代码，则func1和func2不能再并行执行，即将会同步执行
            while(true){
                Thread.sleep(1000);
                System.out.println("func2");
            }
        }
    }

    public void func3() throws InterruptedException {
        while(true){
            Thread.sleep(1000);
            System.out.println("func3");
        }
    }
}

```
　　分别调用资源类的三个工作者线程，其一示例如下：
```
package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class ThreadX extends Thread {
    public Resource r;

    public ThreadX(Resource r) {
        this.r = r;
    }

    @Override
    public void run() {
        try {
            r.func1();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```
　　3)测试类，即调用工作者类。代码示例如下：
```
package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class Handler {

    public static void main(String[] args) {
        Resource resource=new Resource();
        ThreadX x=new ThreadX(resource);
        ThreadY y=new ThreadY(resource);
        ThreadZ z=new ThreadZ(resource);

        Thread t1=new Thread(x);
        Thread t2=new Thread(y);
        Thread t3=new Thread(z);

        t1.start();
        t2.start();
        t3.start();
    }
}

//output：
func1
func2
func3
func1
func2
func3
...
```
　　可知Resource的三个方法可以异步执行。但是如果使用同一个对象加锁的话，则将同步执行。
##### **2.脏读**
　　同步串行化处理数据视为了避免“脏读”，所谓脏读即实际进行操作的数据跟之前“看到的”不一致。示例如下，当两个线程抢夺一个线程，谁先抢到就讲资源命名为自己的名称：
```
package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class ResourceX {

    public String str;

    public ResourceX(String str) {
        this.str = str;
    }

    public String getStr() {
        return str;
    }

    public void setStr(String str) {
        this.str = str;
    }
}


package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class ThreaX extends Thread {

    public ResourceX x;

    public ThreaX(ResourceX x) {
        this.x = x;
    }

    @Override
    public void run() {
        if(x.getStr().equals("shared")){
            x.setStr("XX");
            System.out.println(x.getStr());
        }
    }
}

package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class ThreaY extends Thread {
    public ResourceX x;

    public ThreaY(ResourceX x) {
        this.x = x;
    }

    @Override
    public void run() {
        if(x.getStr().equals("shared")){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(x.getStr());
            x.setStr("YY");
            System.out.println(x.getStr());
        }
    }
}

package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class Handler {

    public static void main(String[] args) {
        ResourceX x=new ResourceX("shared");
        ThreaX tx=new ThreaX(x);
        ThreaY ty=new ThreaY(x);
        new Thread(tx).start();
        new Thread(ty).start();
    }
}
```
　　示例中线程y强多了线程x已经命名的非公共线程；
##### **3.可重入锁**
　　可重入锁即**当一个线程得到一个对象锁后，此时请求此对象锁(或其父类对象锁)时是可以得到该对象的锁的，<font color=red>但是请求其他对象锁是不可以的</font>。**示例如下：
```
package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class Fat {
    String str="fat";

    synchronized public void func(){
        System.out.println(str);
    }
}

package sync;

/**
 * @author 杜艮魁
 * @date 2018/1/16
 */
public class Sub extends Fat {

    synchronized public void func1() throws InterruptedException {
        System.out.println("sub.func1()");
        this.func2();
    }

    synchronized public void func2() throws InterruptedException {
        System.out.println("sub.func2");
        this.func();//fixme “重入父类锁”
    }
}

...
 new Thread(new MyThread(new Sub())).start();
...
```

#### 其他
1. 加锁的对象是可以在加锁代码块中被修改的；
2. 一定要注意加锁的是不是用同一个对象加锁的，还是一个类的多个对象；
3. synchronized特性是不能继承的，即必须在子类中明确指明；