#### 1.引言

传统的Java程序，bean的生命周期非常简单：用`new` 实例化后就可以使用，不再使用时由虚拟机进行垃圾回收。

相比之下Spring的生命周期更加复杂，下图展示了bean 装载到Spring应用上下文中的一个典型的生命周期过程：

![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fu6oq77r09j31kw0ya7v8.jpg)

#### 2. bean生命周期各阶段讲解

##### 2.1 spring 对bean进行实例化；

##### 2.2 Spring将值和bean的引用注入到bean对应的属性中；

##### 2.3 ID/名称: BeanNameAware接口

如果bean实现了BeanNameAware接口，则Spring调用setBeanName()方法。示例如下，就是一个setter方法：

    private String beanName;
    
		public void setBeanName(String beanName) {
        this.beanName = beanName;
    }

##### 2.4 bean 工厂：BeanFactoryAware

如果实现了此接口，则Spring调用...:
```java
 	//也是一个setter方法，但可能涉及其他内容	
    @Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
		if (beanFactory instanceof ConfigurableBeanFactory) {
			ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;
			if (this.beanClassLoader == null) {
				this.beanClassLoader = cbf.getBeanClassLoader();
			}
			this.retrievalMutex = cbf.getSingletonMutex();
		}
	}
```
##### 2.5 上下文 ApplicationContextAware
```java
    private ApplicationContext context;
    private Environment environment;

    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.context = applicationContext;
        this.environment = this.context.getEnvironment();
    }
```
应用上下文往往和Environment有关，[参考博客](https://blog.csdn.net/windsunmoon/article/details/45197361)。**`Environment`** 包含两方面抽象：

1. Profile是一组bean的定义，只有相应的profile被激活的情况下才会有用；

2. 可以方便的访问系统属性、环境变量和自定义属性，也可以加入自定义的property。

##### 2.6 BeanPostProcessor和初始化的InitializingBean接口(aba)

如果bean实现了`InitializingBean`接口或者java 注解创建的bean使用`initMethod =XX`方法声明了初始化的方法，则该方法的`afterPorpertiesSet()`方法；

如果也实现了`BeanPostProcessor`接口，则其`postProcessBeforeInitialization`和`postProcessAfterInitialization`方法将在`afterPorpertiesSet()`执行前后调用`InitializingBean`接口没有实现的话这两个方法也会被调用。

##### 2.7 销毁:DisposableBean和Java配置的destroy-method注解

经过以上6个步骤bean已经驻留在应用上下文中并可被使用，直到应用上下文被销毁。

如果bean实现了`DisposableBean`接口或者Java配置是`@bean`注解使用了`destroy-method`属性，则`DisposableBean`接口中的`destroy()`方法会被调用。

#### 3.代码示例

##### 3.1常用生命周期相关接口





##### 3.2 完整生命周期接口实现示例

```java
@Service//表明这是一个bean，如果xml或javaConfig中配置了扫描路径就会初始化此bean
public class BeanLifeCycleTest implements BeanNameAware, BeanFactoryAware, ApplicationContextAware
        , BeanPostProcessor, InitializingBean, DisposableBean {

    private BeanFactory beanFactory;

    private ApplicationContext applicationContext;

    private Environment environment;


    @Autowired
    private AlgorithmCacheClient autowiredBean;

    @Override
    public void setBeanName(String name) {
        System.out.println("1：初始化实例");
        System.out.println("2：将引入的注入赋值给相应的属性,autowiredBean's class"+autowiredBean.getClass().getSimpleName());

        System.out.println("3:setBeanName(BeanNameAware)");
        //todo
        name="beanName";
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.println("4:setBeanFactory(BeanFactoryAware)");
        this.beanFactory=beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.print("5:setApplicationContext(ApplicationContextAware)");
        this.applicationContext=applicationContext;
        environment=applicationContext.getEnvironment();
        System.out.println(",env"+environment.getActiveProfiles());
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        //todo:bean是哪里来的
        System.out.println("6:beanName:"+beanName+",beanObject:"+bean.getClass()+".postProcess-Before-Initialization");
        return null;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("7,afterPropertiesSet[InitializingBean]");
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        //todo:bean是哪里来的
        System.out.println("8:beanName:"+beanName+",beanObject:"+bean.getClass()+".postProcess-Before-Initialization");
        return null;
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("END");
    }
}

output:

1：初始化实例
2：将引入的注入赋值给相应的属性,autowiredBean's classAlgorithmCacheClient
3:setBeanName(BeanNameAware)
4:setBeanFactory(BeanFactoryAware)
5:setApplicationContext(ApplicationContextAware),env[Ljava.lang.String;@61d9dd15
7,afterPropertiesSet[InitializingBean]
6:beanName:getExternalFeature,beanObject:class com.sankuai.meituan.waimai.algorithm.GetExternalFeature.postProcess-Before-Initialization
8:beanName:featureFetchThriftService,beanObject:class com.sun.proxy.$Proxy31.postProcess-Before-Initialization
END

```