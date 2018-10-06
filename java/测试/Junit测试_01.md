- [源码参考](https://github.com/dugenkui03/incubator-dubbo/blob/3.x-dev-comment/dubbo-filter/dubbo-filter-cache/src/test/java/org/apache/dubbo/cache/filter/CacheFilterTest.java)


##### 1. 测试内容

测试内容要周全，比如有参无参、异常和null四种基本情况。

##### 2. @Parameters

使用不同的参数进行多次测试时可以使用:`@Parameters`，基本方式如下：

```java
//Parameterized.class
@RunWith(Parameterized.class)
public class DemoTest{
    private String name;
    private Integer age;
    
    public DemoTest(String name,Integer age){
        this.name=name;
        this.age=age;
    }
    
    /**
     *1. public static Collection funcName()：公共静态、返回值为Collectoin、和无参函数；
     *2. 数组内容必须与构造参数对齐。
     **/
    @Parameters
    public static List<Object[]> info(){
        return Arrays.asList(new Object[][]{
            {"zhao",1},
            {"qian",2},
            {"sun",3}
        });
    }
    
    ...
}
```

##### 3. 使用@Before初始化变量

@Test每次运行时都会运行一次@Before中的代码

##### 4. given(obj.func(paramObj)).willReturn(Object) 和 mock
 - import static org.mockito.BDDMockito.given 中，given(obj.func()).willReturn()
  设置obj以paramObj为参数时调用func()时的返回值。
 - import static org.mokito.Mockito.mock:mock对象
 
 private Invoker<?> invoker=mock(Invoker.class);
 ...
 given(invoker.invoke(1)).willReturn(new Integer(1));
 
  
 