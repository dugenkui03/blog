#### 一.分流和负载均衡区别

**<font color=red>结论：AB实验分流不能丢失策略，分流比要完美趋近权重。</font>**

**负载均衡**只要将流量根据权重打到不同的机器即可，所有机器的业务逻辑都是一样的。

**AB实验分流**则需要：
1. 将同一个请求多次访问时打到同一个策略，保证请求不丢失策略。比如用户访问一个页面时不能交替出现不同的展示；
2. 同**负载均衡**,保证流量比完美趋近于权重。

综上**AB实验需要保证<font color=red>不丢失策略</font><font color=blue>流量分配无限接近与权重比。</font>**

#### 二.常用负载均衡优缺点

<font color=red>**结论：源信息hash算法是唯一不丢失策略的算法，但是负载均衡和源数据信息和哈希算法有很大关系。**</font> 因为分流策略是工程方引入的jar包，平台也需要提供调试功能，因此也不能使用统一的缓存管理。


常用负载均衡算法有**轮询法、随机法、源地址哈希、一致性哈希和最少活跃调用**五种思路。其中 [dubbo](https://github.com/apache/incubator-dubbo/tree/master/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance)实现除了源地址哈希的其他四种。其详细分析见另一片博文，此处重点讲源地址哈希算法：

1. 轮询法：最趋于平衡的负载均衡算法，但是修改指针时需要加锁，性能较低；
2. 随机发：请求数量越高平衡性越趋近于轮询，一般使用随机数或者`System.identityHashCode(obj)`；
3. 源地址哈希：**<font color=red>唯一不丢失策略的算法</font>，但是负载均衡和源数据信息和哈希算法有很大关系**，AB实验分流一般采用此方法；
4. 一致性哈希算法：todo
5. 最少获取调用：todo


#### 三.几种分流策略的标准差比较

使用轮询和随机路由做基准，然后对比uuid简单hash、考虑高位的hash、md5算法以及md5后hash，**<font color=red>经测md5后hash效果最好</font>**。


37617条数据，分成100个桶，几种方法标准差(程序输出)如下：
```
期望值：expect：376.17
轮询：0.375632799419859	base
随机路由：17.838192172975376	randomRoutain

//如下可知md5然后哈希的效果最好
uuid简单哈希：19.11337489822245	simpleHash
考虑高位的uuid哈希： 19.753002303447445	bitHash from HashMap:
md5值转数字取模：76.02381929369243	md5Hash:
md5值求哈希： 18.963678440640148	md5ThenHash
```

```java

/**
 * @Description 标准差衡量数据总体波动范围(方差描述离散程度)
 * fixme:一千万次调用，一百个桶，每个桶期望值十万；
 * @Date 2018/9/8 下午4:48
 * -
 * @Author dugenkui
 **/

public class SourceHashBalanceTest {
    /**
     * 获取uuid数据集合
     */
    static List<String> uuidList = new ArrayList<>();
    static {
        try {
            File file = new File("/Users/moriushitorasakigake/Desktop/uuid1.txt");
            BufferedReader fileReader = new BufferedReader(new FileReader(file));
            String tmp;
            while ((tmp = fileReader.readLine()) != null) {
                uuidList.add(tmp);
            }
        } catch (Exception e) {
            System.out.println("请在指定目录添加样本数据");
        }
    }

    /**
     * 实验次数，分桶和每个桶期望值
     */
    static final long EXP_COUNT = uuidList.size();
    static final long MODEL = 100;
    static final double AVG_COUNT = (double)EXP_COUNT / MODEL;


    //基准测试的标准差：证明样本数据每个值完全不同而且变化时标准差为0.0
    static double baseTest() {
        Map<Integer, Integer> pointCount = new HashMap();
        for (int i = 0; i < EXP_COUNT; i++) {
            int pos = i % 100;
            pointCount.compute(pos, (k, v) -> v == null ? 1 : ++v);
        }

        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }


    /**
     * 随机方法的误差：理论上请求量越大误差越小
     */
    static double randomRoutain() {
        Random rand = new Random(System.currentTimeMillis());
        Map<Integer, Integer> pointCount = new HashMap();
        for (int i = 0; i < EXP_COUNT; i++) {
            int hash = Math.abs(rand.nextInt() % 100);
            pointCount.compute(hash, (k, v) -> v == null ? 1 : ++v);
        }
        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }

    /**
     * 字符串简单哈希求值
     *
     * @return
     */
    static double simpleHash() {
        Map<Integer, Integer> pointCount = new HashMap();
        for (String uuid : uuidList) {
            int hash = Math.abs(uuid.hashCode() % 100);
            pointCount.compute(hash, (k, v) -> v == null ? 1 : ++v);
        }
        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }

    /**
     * 字符串注意高位影响的哈希(参考HashMap)
     *
     * @return
     */
    static double bitHash() {
        Map<Integer, Integer> pointCount = new HashMap();
        for (String uuid : uuidList) {
            int hash = Math.abs((uuid.hashCode() ^ (uuid.hashCode() >>> 16)) % 100);
            pointCount.compute(hash, (k, v) -> v == null ? 1 : ++v);
        }
        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }


    /**
     * 字符串求MD5值取模
     *
     * @return
     */
    static double md5Hash() {
        Map<Integer, Integer> pointCount = new HashMap();
        for (String uuid : uuidList) {
            int md5Num=Integer.parseInt(DigestUtils.md5DigestAsHex(uuid.getBytes()).substring(30),16);
            int hash = Math.abs(md5Num % 100);
            pointCount.compute(hash, (k, v) -> v == null ? 1 : ++v);
        }
        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }

    /**
     * 字符串求MD5值后在hash
     *
     * @return
     */
    static double md5ThenHash() {
        Map<Integer, Integer> pointCount = new HashMap();
        for (String uuid : uuidList) {
            int hash = Math.abs(DigestUtils.md5DigestAsHex(uuid.getBytes()).hashCode() % 100);
            pointCount.compute(hash, (k, v) -> v == null ? 1 : ++v);
        }
        //标准差计算公式
        double tmpPowSum = pointCount.values().stream().collect(Collectors.summingDouble(x -> Math.pow((x - AVG_COUNT), 2)));
        return Math.sqrt(tmpPowSum / MODEL);
    }

    public static void main(String[] args) {
        System.out.println("expect："+AVG_COUNT);
        System.out.println(baseTest() + "\tbase");
        System.out.println(randomRoutain() + "\trandomRoutain");
        System.out.println(simpleHash() + "\tsimpleHash");
        System.out.println(bitHash() + "\tbitHash from HashMap:");
        System.out.println(md5Hash() + "\tmd5Hash:");
        System.out.println(md5ThenHash() + "\tmd5ThenHash");

    }
}

```


#### 四.常用负载均衡算法的实现

见另一篇文章，主要讨论  [dubbo](https://github.com/apache/incubator-dubbo/tree/master/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance)中实现的集中负载均衡算法。




