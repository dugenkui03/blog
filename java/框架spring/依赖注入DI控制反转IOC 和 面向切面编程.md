
POJO： 简单的plain 老式old java 对象Object
##### 1. 依赖注入DI

任何一个应用都有两个以上的类组成，这些类相互协作完成特定的任务逻辑。传统做法是**每个对象负责管理与自己协作的对象的引用(所依赖的对象)，但是这将导致高度耦合**。

编写代码应该高内聚、低耦合，耦合会导致难以复用、理解。

依赖注入DI又叫控制反转，指的是由**spring负责对象的第三方组件的创建和管理工作。对象无须创建管理他们依赖的引用，依赖关系自动注入他们需要的对象中**。

###### 1.1 依赖注入的三种方式

- 构造器注入constructor injection：示例中`BraveKnight`没有与特定的Quest实现发生耦合，通过接口来实现依赖关系，能够用不同的具体实现进行替换。

```java
public class BraveKnight implements Knight{
    private Quest quest;
    
    public BraveKnight(Quest quest){
        this.quest=quest;
    }
    
    public void embarkOnQuest(){
        quest.embark;
    }
}

@Configuration//定义bean装配类
public class KnightConfig{
    
    //方法名默认就是Bean的名称，该方法返回值就是Bean的对象
    @bean
    public Knight knight(){
        return new BraveKnight(quest());
    }
    
    @bean
    public Quest quest(){
        return new SlayDragonQuest(System.out);
    }
}
```
- `@Autowired`接口注入：注入的类必须有相关注释
- `Setter`方法注入；



###### 1.2 装配

<font color=red> **创建应用组件之间协作的行为称为装配**</font>。装配最方便的方式是自动扫描包：**在spring mvc 配置了这个标签后，spring会自动扫描`base-package`及其子包的java文件，如果扫描到有`@Component @Controller@Service`等这些注解的类，则把这些类注册为bean**:
```
<context:component-scan base-package="package.name" annotation-config="true"/>
```

也可是使用java配置的方式：
```java

@Configuration//定义bean装配类
public class KnightConfig{
    
    //方法名默认就是Bean的名称，该方法返回值就是Bean的对象
    @bean
    public Knight knight(){
        return new BraveKnight(quest());
    }
    
    @bean
    public Quest quest(){
        return new SlayDragonQuest(System.out);
    }
}

```

##### 2. 面向切面编程AOP

DI(IOC)实现松耦合，面向切面编程把应用各处功能分离出来形成可重用的组件。

我们总希望每个类专注于自己的核心逻辑，但是也总需要使用日志子类的服务——这些系统服务通常称为**横切关注点**，因为他们会跨越系统的多个组件。

###### 2.1 切点的配置方式

1. 首先要将横切关注点类声明为一个bean，或者直接用`@Aspect`;
2. 创建横切面；

**切点表达式**
![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fr6cweuuwvj30fy06fdi4.jpg)

- 如上切点表达式表示了当`* com.sif.bnea.Singer.perform(..)`方法的任意重载(包括参数和返回值)执行时触发通知调用。

- 表达式后边添加`&&within(packageName.*)`表示配置的切点仅仅匹配packageName包。

- bean(XXX)


横切面的创建示例：
```java

@Aspect//将当前类标识为一个切面
public class Audience{
    
    //定义切点及其前置动作
    @Before("execution(** package.Perform.perform(..))")
    public void beforeAction(){
        sout("hahaha");
    }
    
    @AfterReturning("execution(** package.Perform.perform(..))")
    public void after(){
        sout("wondful");
    }
    
    @AfterThrowing("execution(** package.Perform.perform(..))")
    public void after demandRefund(){
        sout("bad");
    }
    
    @Around
}

```

##### 3.常用注解

- `@Controller`
- `@RequestMapping(value = "/simple",method = RequestMethod.POST)
`：映射的路径和映射到的请求类型；
- `@Configuration @bean`：装配配置方式
- `@Autowired`：定义在要注入的接口上（但是接口不用`@Service`注释，实现接口的类需要`@Service`注释；
-  `@RequestParam(name="nameid") String id`:也是一种注入，将表单提交参数名与方法参数对齐；
-  `@RequestBody`:将Request请求的body部分数据解析到参数指定的对象上；使用系统默认配置的HttpMessageConverter进行解析；
- `@Responsebody`：将方法返回结果返回到请求对应的http响应中而不会被解析为跳转路径；
- `@PathVariable`:如果`@RequestMapping("/users/{username}")`使用了变量，则方法中可以使用`String userProfile(@PathVariable("username") String username)`;
- `@Transactional` 只能应用到 public 方法才有效;
- `@Scheduled(cron = "0/1 * * * * ?")`：实现定时任务；
- `@Entity @Table`:@Entity注释指名这是一个实体Bean，@Table注释指定了Entity所要映射带数据库表，其中@Table.name()用来指定映射表的表名。如果缺省@Table注释，系统默认采用类名作为映射表的表名；
- `@Aspect`：是把当前类标识为一个切面:`@Befroe @AfterReturn @AfterThrowing @Around`;


