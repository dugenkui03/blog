##### 1. 业务方调用算法参数

入口。使用`AlgorithmInvoker`的`public BaseOutput invoke(String algorithmKey,BaseInput  algorithmInput)`方法。

输入参数和返回值内容如下：
```java
/*-----------------------------输入--------------------------------*/

//输入参数有特征Feature、维度DimeDimenssion、参数Params和实验参数 Map<String,Object> expParam 四种：
public class BaseInput {
    private Features features;
    private Dimensions dimensions = new DefaultDimensions();
    private Params params;
    private Map<String,Object> expParam;
}

//特征变量:另一个ExternalFeatures支持自定义Map<String,String>特征kv
public class DefaultFeatures extends Features {
     // dimensionType --> dimensionValue --> featureKey --> featureValue
    private Map<String, Map<String, Map<String, FeatureValue>>> features = new HashMap<>();
}

public class FeatureValue{
    private String value;
    private boolean defaultValue;
    private String timestamp;
}

//维度变量：因为一个维度可以对应多个值，所以v是List
public class DefaultDimensions implements Dimensions {
    private Map<String, List<String>> values = new HashMap<>();
		//以下函数将DimensionType的value和对应的维度值放进valueMap中
    public void addArea(long areaId)...
    public void addPoi(long poiId)...
    public void addUuid(String uuid) ...
    public void addOrder(long orderId) ...
    public void addUser(long userId) ...
    public void addQuery(String query) ...
}

public enum DimensionType {
    DEFAULT(0, "default"),
    AREA(1, "area"),
    POI(2, "poi"),
    UUID(3, "uuid"),
    ORDER(4, "order"),
    USER(5, "user"),
    QUERY(6, "query"),
    EXPERIMENT(99, "experiment");
}

//Params参数
public class Params {
    private final Map<String, String> paramMap;
    private final Map<String, List<String>> paramMapList;
}

//实验参数
private Map<String,Object> expParam;



/*-----------------------------输出--------------------------------*/

//主要保存返回给业务方的 计算结果 和 算法调用详情——（比如调用algorithmKey和命中的versionKey，以及调用的维度和Params参数。算法方返回值不一定与调用值相同，只是类型相同）
public class BaseOutput {
    private Object result;
    private InvokeDetail invokeDetail;
}

public class InvokeDetail {
    private String algorithmKey;
    private String versionKey;
    private Dimension dimension;
    private Params params;
}
```

##### 2.调用算法：输入定位算法实现的algorithmKey和输入参数BaseInput

```java
    public O invoke(String algorithmKey, I algorithmInput) {
          	//routingMap就是shuffleMap，用于实验分流，原始值保存在Dimensions中：
            Map<String, String> routingMap = algorithmInvokerService.getRoutingMapByAlgorithmInput(algorithmInput);
          
      			/*getExperiment调用链：根据根据算法Key查找是否有上线的实验，有则根据routingMap和ExpParam中的数据判断是否符合准入机制或命中白名单、根据routingMap进行路由
             * algorithmInvokerService.getExperiment(algorithmKey, routingMap, algorithmInput.getExpParam())->
             * List<Experiment> experimentList = experimentManger.getExperiments(expParam)——>  =====================命中的实验是返回链表的第一个实验
             * List<Experiment> exps = experimentCacheService.getExperimentList(expParam)->
             * Out out = this.engine.cal(in);
             * 参数详解：
             * ExpParam:String sceneKey;String uuid;long userId;String appVersion;Map<String, Object> extendParam;
             * Out: List<Exp> exps; In:Map<String, Object> param,String key;
             */
      			Experiment exp = algorithmInvokerService.getExperiment(algorithmKey, routingMap, algorithmInput.getExpParam());
            
      			//如果命中某个实验，调用某个实验对输入进行计算并返回结果：算法实验exp.getExpKey()返回的是算法版本；
            if (exp != null && StringUtils.isNotBlank(exp.getExpKey())) {
            		return invoke(algorithmKey, null, algorithmInput, exp.getExpKey(), exp.getParam());
            }
          	
      			//如果没有配置实验，则调用算法默认版本进行计算并返回结果
            String versionKey = algorithmCacheClient.getVersionKey(algorithmKey);
            O output = invoke(algorithmKey, null, algorithmInput, versionKey);
            return output;
    }

//分流类型
public class DimensionToRountinKey {
    public static Map<String, String> dimensionRountinKeyMap = new HashMap<>();
    static {
        dimensionRountinKeyMap.put("user","userId");
        dimensionRountinKeyMap.put("uuid","uuid");
        dimensionRountinKeyMap.put("poi","poiId");
        dimensionRountinKeyMap.put("order","orderId");
        dimensionRountinKeyMap.put("area","areaId");
    }
}
```
##### 3.获取algorithmKey对应的实验配置，分流

 简介见第二小节代码注释。具体代码调用栈如下：

```java
/*-------------------------------------------launcher--------------------------------------------------------------------------------*/

//AlgorithmInvoker<I extends BaseInput, O extends BaseOutput>	判断算法是否设置了上线的实验组，有则命中其中实验，无则流量命中默认算法 
	public O invoke(String algorithmKey, I algorithmInput) {
      			//routingMap就是shuffleMap，用于实验分流，原始值保存在Dimensions中：
            Map<String, String> routingMap = algorithmInvokerService.getRoutingMapByAlgorithmInput(algorithmInput);
						//1.
      			Experiment exp = algorithmInvokerService.getExperiment(algorithmKey, routingMap, algorithmInput.getExpParam());
            
      			//如果命中某个实验，调用某个实验对输入进行计算并返回结果：算法实验exp.getExpKey()返回的是算法版本；
      			//如果没有配置实验，则调用算法默认版本进行计算并返回结果
    }

//AlgorithmInvokerService
    public Experiment getExperiment(String algorithmKey, Map<String, String> shuffleKeys, Map<String, Object> expParamMap) {
      	//将分流shuffleKeys也是routingMap和expParamMap中UUID、USER_ID、POI_ID、ORDER_ID、AREA_ID封装进ExpParam的sceneKey、uuid、userId和Map<String, Object> extendParam中
        ExpParam expParam = initExpParam(algorithmKey,shuffleKeys,expParamMap);
				
      	
        List<Experiment> experimentList = experimentManger.getExperiments(expParam);
        if(CollectionUtils.isNotEmpty(experimentList)){
            return experimentList.get(0);
        }
        logger.info("getExperiment cant match exp ,algorithmKey:{},shuffleKeys:{}", algorithmKey, shuffleKeys);
        return null;
    }

/*------------------------------------------------------exp_core--------------------------------------------------------------------------------*/

//ExperimentManager:仅此一个调用方法作为 launcher的入口
    public List<Experiment> getExperiments(ExpParam expParam) {
        List<Experiment> exps = experimentCacheService.getExperimentList(expParam);
        return exps;
    }

  /**ExperimentCacheService类
   * 
   *1.checkAndPullSceneConfig:ExperimentCacheService类保存当前业务方调用过的实验场景key`private Set<String> sceneKeys = Sets.newConcurrentHashSet()`，
   *	定时更新配置，并将上线的场景配置<sceneKey,ExpScene>更新到ExpEngine的缓存中，ExpEngine负责具体的分流计算。此方法的作用是：1.如果调用的algorithmKey/sceneKey
   *	没有在ExperimentCacheService缓存汇中、并将拉取当前sceneKey对应实验配置放入ExpEngine的任务放入任务队列。
   * 
   *2.构造输入参数
   *
   *3.传入sceneKey和计算参数，然后根据sceneKey获取激活的场景列表，通过准入条件、白名单命中和分流后输出流量命中的实验列表：Out: List<Exp> exps;
   */
    public List<Experiment> getExperimentList(ExpParam expParam) {
      	//1.
        checkAndPullSceneConfig(expParam.getSceneKey());
				
      	//2.构造输入In:String key, Map<String, Object> param:
        ExpEngine.In in = new ExpEngine.In();
      	Map<String, Object> paramMap = initExpParamMap(expParam);
        in.setKey(expParam.getSceneKey());
        in.setParam(paramMap);
				
      	//计算命中的实验列表：todo如果1处任务没有更新当前sceneKey对应的ExpEngine缓存，则此处不能命中任何实验，即配置实验后不能立即生效，可使用FutureTask做缓存的value来改进
        ExpEngine.Out out = engine.cal(in);
      	//其实返回之前还有判空和转换格式的步骤
        return out.getExps();
    }		


////////////////ExperimentCacheService类定时任务更新缓存sceneKey对应的实验场景配置，将上线的更新到ExpEngine缓存<String sceneKey,ExpScene config>缓存中，ExpEngine负责分流。
///////////////每次调用新sceneKey都会将拉取其实验配置加入到工作队列。


--->ExperimentManager: public List<Experiment> getExperiments(ExpParam expParam);//todo：这里应该返回命中的某个实验，而不应该让Launcher选择命中的实验列表中的第一个实验。
--->AlgorithmInvokerService: Experiment getExperiment(String algorithmKey, Map<String, String> shuffleKeys, Map<String, Object> expParamMap):todo：同上
```

##### 4.封装参数并调用算法

```java
  /**
  	*
    *
    */
	private O invoke(String algorithmKey, Dimension dimension, I algorithmInput, String versionKey, Map<String, String> expParam) {
        //如果命中空白实验组
        if ((algorithmKey + LauncherConstant.EMPTY_VERSION_SUFFIX).equals(versionKey)) {
            return emptyVersionOutput(algorithmKey, algorithmInput);
        }

        // 调用算法
        IAlgor<I, O> algor = algorithmCacheClient.getAlgorBean(versionKey);
        if (algor == null) {
            logger.error("BeanDefinitionNotFoundException versionKey:{}", versionKey);
            throw new BeanDefinitionNotFoundException("算法Bean未定义, key= " + versionKey);
        }

        Transaction fetchTransaction = Cat.newTransaction(CatCons.ALGORITHM_INVOKER, versionKey + "_" + CatCons.FETCH_FEATURE);
        try {
            fetchFeatureService.fetchFeatureValues(algorithmKey, algorithmInput, versionKey);
            fetchTransaction.setStatus(Transaction.SUCCESS);
        } catch (Exception e) {
            logger.error("fetchFeatureValues Exception. versionKey:{}", versionKey, e);
            fetchTransaction.setStatus(e);
            algorithmInput.setFeatures(null);
            Cat.logError(e);
        } finally {
            fetchTransaction.complete();
        }

        //拼装算法参数
        Params params = constructAlgorithmParams(algorithmKey, versionKey, expParam);

        if (algorithmInvokeInterceptors != null) {
            for (AlgorithmInvokeInterceptor interceptor : algorithmInvokeInterceptors) {
                Transaction t = Cat.newTransaction(CatCons.ALGORITHM_INVOKER,
                        interceptor.getClass().getSimpleName());
                try {
                    interceptor.preProcess(algorithmKey, dimension, algorithmInput, params.getParamMap());
                    t.setStatus(Transaction.SUCCESS);
                } catch (Exception e) {
                    t.setStatus(e);
                    Cat.logError(e);
                    throw e;
                } finally {
                    t.complete();
                }
            }
        }

        O algorithmOutput = algorithmInvokerService.invokeCompute(algor, versionKey, algorithmInput, params);
        if (algorithmOutput == null) {
            logger.warn("algorithmOutput is null,algorithmKey:{},versionKey:{}", algorithmKey, versionKey);
            algorithmOutput = (O) new BaseOutput();
        }


        // 设置调用算法详情
        InvokeDetail invokeDetail = new InvokeDetail.Builder(algorithmKey, dimension)
                .params(params)
                .versionKey(versionKey).build();
        algorithmOutput.setInvokeDetail(invokeDetail);

        return algorithmOutput;
    }
```

##### 4. 其他

1. 改进1:实验立即生效public volatile static Map<String, ExpScene> sceneCache = new ConcurrentHashMap<>();使用FetureTask;

2. 默认版本和实验配置定时拉取：想办法触发。


##### XX.关于分流

1. sceneKey是客户端调用AB分流方法的重要参数，创建后不可修改；

2. groupKey,修改这个参数会重新实验打散流量—流量hash时的参数；

3. 无论是默认还是自定义分流字段，都需要确保在业务方调用的特征的dimension或算法的params中存在；

4. 所有配置的条件之间都是逻辑与的关系，同时满足所有条件的流量才会进入实验。