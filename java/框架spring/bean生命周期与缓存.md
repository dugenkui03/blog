#### 1.引言

接入方调用函数`BaseOutput invoek(String calcutorType,BaseInput in,Map param)`，指定某种类型的计算器对输入+参数进行计算。

调度方(服务方)根据`calcutorType`找到合适的计算器进行计算，并返回结果。实现调度的策略是 **<font color=red> 将计算器类型和计算器的bean维护在map中。</font>**

#### 2.结构设计

加载`bean`时，计算器bean缓存bean类实现<font color=red> `ApplicationContextAware,InitializingBean`</font>：

1. 第一个接口可用于获取上下文，

2. 第二个接口待所有bean加载成功后，扫描所有bean，得到实现计算器接口的bean，放在其缓存map中

调用方`Invoke`传入计算类型`type`和计算参数`BaseInput,Map`。计算器缓存bean根据入参type类型，调用相应的计算器方法，返回结果。

#### 3.代码实现

- 最重要的放前边(调度器、计算器bean缓存map)

```java
@Service
public class CalCutorCache implements ApplicationContextAware,InitializingBean {

    private ApplicationContext applicationContext;

    /**
     * key:calType注解；value：计算器bean实例；
     */
    private Map<String, Calcutor> calcutors;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext=applicationContext;
    }

    @Override
    public void afterPropertiesSet() {
        Map<String, Calcutor> beanNameAndBean = applicationContext.getBeansOfType(Calcutor.class);

        calcutors=new HashMap<>();
        for (Map.Entry<String, Calcutor> en:beanNameAndBean.entrySet()) {
            Class<? extends Calcutor> calService = en.getValue().getClass();
            //获取类上某种类型的注解
            CalcutorType calcutorType=calService.getAnnotation(CalcutorType.class);
            calcutors.put(calcutorType.type(),en.getValue());
        }
    }

    public Calcutor getCalcutor(String type){
        return calcutors.get(type);
    }
}
```

- 调用方(业务方)

```java
@Service
public class Invoker<I extends BaseInput,O extends BaseOutput> {

    @Autowired
    private CalCutorCache cutorCacheClient;

    public O invoke(String calType, I input, Map param){
        Calcutor calcutor=cutorCacheClient.getCalcutor(calType);
        BaseOutput out= calcutor.compute(input,param);

        return (O)out;
    }
}
```

- 基本输入输入和计算器接口

```java
@AllArgsConstructor
public class BaseInput {
    Object basicParamX;
}

@AllArgsConstructor
public class BaseOutput {
    Object basicOutputX;
}
```

- 计算器实现类(加法、乘法)

```java
@CalcutorType(type="add")
@Service
public class AddCalcutor implements Calcutor {
    @Override
    public BaseOutput compute(BaseInput input, Map param) {
        System.out.println("baseIntpu's type"+input.getClass().getSimpleName());

        int sum=0;
        for (Object ele:param.values()) {
            int factor=NumberUtils.toInt(ele.toString(),0);
            sum+=factor;
        }

        return new BaseOutput(sum);
    }
}

@CalcutorType(type="multi")
@Service
public class MultiCalcutor implements Calcutor {

    @Override
    public BaseOutput compute(BaseInput input, Map param) {
        System.out.println("baseIntpu's type"+input.getClass().getSimpleName());
        int prod=1;
        for (Object ele:param.values()) {
            int factor= NumberUtils.toInt(ele.toString(),0);
            prod*=factor;
        }

        return new BaseOutput(prod);
    }
}

```

- 计算器类型注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface CalcutorType {
    String type() default "";
}

```