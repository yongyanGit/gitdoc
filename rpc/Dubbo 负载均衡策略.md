### Dubbo 负载均衡策略

#### 一、dubbo的LoadBalance接口

dubbo的负载均衡策略，主体向外暴露出来是一个接口，名字叫做loadBlace,位于com.alibaba.dubbo.rpc.cluster包下，很明显根据包名就可以看出它是用来管理集群的：

这个接口就一个方法，select方法的作用就是从众多的调用的List选择出一个调用者，Invoker可以理解为客户端的调用者，dubbo专门封装一个类来表示，URL就是调用者发起的URL请求链接，从这个URL中可以获取很多请求的具体信息,Invocation表示的是调用的具体过程

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

   /**
    * select one invoker in list.
    * 
    * @param invokers invokers.
    * @param url refer url
    * @param invocation invocation.
    * @return selected invoker.
    */
    @Adaptive("loadbalance")
   <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```

#### 二、AbstractLoadBalance

在 Dubbo 中，所有负载均衡实现类均继承自 AbstractLoadBalance，该类实现了 LoadBalance  接口方法，并封装了一些公共的逻辑。所以在分析负载均衡实现之前，先来看一下 AbstractLoadBalance  的逻辑。首先来看一下负载均衡的入口方法 select，如下：

```java
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    if (invokers == null || invokers.size() == 0)
        return null;
    //如果invokers 列表中仅有一个Invoker,直接返回，无需进行负载均衡
    if (invokers.size() == 1)
        return invokers.get(0);
    //调用 doSelect 方法进行负载均衡，该方法为抽象方法，由子类实现
    return doSelect(invokers, url, invocation);
}

protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
```

select 方法的逻辑比较简单，首先会检测 invokers 集合的合法性，然后再检测 invokers 集合元素数量。如果只包含一个  Invoker，直接返回该 Inovker 即可。如果包含多个 Invoker，此时需要通过负载均衡算法选择一个  Invoker。具体的负载均衡算法由子类实现，接下来章节会对这些子类进行详细分析。

AbstractLoadBalance 除了实现了 LoadBalance 接口方法，还封装了一些公共逻辑 —— 服务提供者权重计算逻辑。具体实现如下

```java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    //从 url 中获取 weight 配置值
    int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
    
    if (weight > 0) {
     //获取服务提供者启动时间戳   
     long timestamp = invoker.getUrl().getParameter(Constants.TIMESTAMP_KEY, 0L);
   if (timestamp > 0L) {
       // 计算服务提供者运行时长
      int uptime = (int) (System.currentTimeMillis() - timestamp);
       // 获取服务预热时间，默认为10分钟
      int warmup = invoker.getUrl().getParameter(Constants.WARMUP_KEY, Constants.DEFAULT_WARMUP);
       //如果服务运行时间小于预热时间，则重新计算服务权重，即降权
      if (uptime > 0 && uptime < warmup) {
          // 重新计算服务权重
         weight = calculateWarmupWeight(uptime, warmup, weight);
      }
   }
    }
   return weight;
}

static int calculateWarmupWeight(int uptime, int warmup, int weight) {
   // 计算权重，下面代码逻辑上形似于 (uptime / warmup) * weight。
   // 随着服务运行时间 uptime 增大，权重计算值 ww 会慢慢接近配置值 weight
   int ww = (int) ( (float) uptime / ( (float) warmup / (float) weight ) );
   return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```

上面是权重的计算过程，该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。服务预热是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。关于预热方面的更多知识，大家感兴趣可以自己搜索一下。

关于 AbstractLoadBalance 就先分析到这，接下来分析各个实现类的代码。首先，我们从 Dubbo 缺省的实现类 RandomLoadBalance 看起。

### 2.1 RandomLoadBalance

RandomLoadBalance 是加权随机算法的具体实现，它的算法思想很简单。

假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3,  2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10)  区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10)  之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。

权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。

比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。

以上就是 RandomLoadBalance 背后的算法思想，比较简单，不多说了，下面开始分析源码。

```java
public class RandomLoadBalance extends AbstractLoadBalance {
public static final String NAME = "random";

private final Random random = new Random();

@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    int length = invokers.size();
    int totalWeight = 0;
    boolean sameWeight = true;
    // 下面这个循环有两个作用，第一是计算总权重 totalWeight，
    // 第二是检测每个服务提供者的权重是否相同，若不相同，则将 sameWeight 置为 false
    for (int i = 0; i < length; i++) {
        int weight = getWeight(invokers.get(i), invocation);
        // 累加权重
        totalWeight += weight;
        // 检测当前服务提供者的权重与上一个服务提供者的权重是否相同，
        // 不相同的话，则将 sameWeight 置为 false。
        if (sameWeight && i > 0
                && weight != getWeight(invokers.get(i - 1), invocation)) {
            sameWeight = false;
        }
    }
    
    // 下面的 if 分支主要用于获取随机数，并计算随机数落在哪个区间上
    if (totalWeight > 0 && !sameWeight) {
        // 随机获取一个 [0, totalWeight) 之间的数字
        int offset = random.nextInt(totalWeight);
        // 循环让 offset 数减去服务提供者权重值，当 offset 小于0时，返回相应的 Invoker。
        // 还是用上面的例子进行说明，servers = [A, B, C]，weights = [5, 3, 2]，offset = 7。
        // 第一次循环，offset - 5 = 2 > 0，说明 offset 肯定不会落在服务器 A 对应的区间上。
        // 第二次循环，offset - 3 = -1 < 0，表明 offset 落在服务器 B 对应的区间上
        for (int i = 0; i < length; i++) {
            // 让随机值 offset 减去权重值
            offset -= getWeight(invokers.get(i), invocation);
            if (offset < 0) {
                // 返回相应的 Invoker
                return invokers.get(i);
            }
        }
    }
    
    // 如果所有服务提供者权重值相同，此时直接随机返回一个即可
    return invokers.get(random.nextInt(length));
}
}
```
RandomLoadBalance 的算法思想比较简单，在经过多次请求后，能够将调用请求按照权重值进行“均匀”分配。当然  RandomLoadBalance 也存在一定的缺点，当调用次数比较少时，Random  产生的随机数可能会比较集中，此时多数请求会落到同一台服务器上。这个缺点并不是很严重，多数情况下可以忽略。RandomLoadBalance  是一个简单，高效的负载均衡实现，因此 Dubbo 选择它作为缺省实现。

### 2.2 LeastActiveLoadBalance

LeastActiveLoadBalance  翻译过来是最小活跃数负载均衡，所谓的最小活跃数可理解为最少连接数。即服务提供者目前正在处理的请求数（一个请求对应一条连接）最少，表明该服务提供者效率高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。

在具体实现中，每个服务提供者对应一个活跃数  active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快。此时这样的服务提供者能够优先获取到新的服务请求，这就是最小活跃数负载均衡算法的基本思想。

除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。

举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的可能性就越大。如果两个服务提供者权重相同，此时随机选择一个即可。

关于 LeastActiveLoadBalance 的背景知识就先介绍到这里，下面开始分析源码。

```java
public class LeastActiveLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "leastactive";

    private final Random random = new Random();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();
        // 最小的活跃数
        int leastActive = -1;
        // 具有相同“最小活跃数”的服务者提供者（以下用 Invoker 代称）数量
        int leastCount = 0; 
        // leastIndexs 用于记录具有相同“最小活跃数”的 Invoker 在 invokers 列表中的下标信息
        int[] leastIndexs = new int[length];
        int totalWeight = 0;
        // 第一个最小活跃数的 Invoker 权重值，用于与其他具有相同最小活跃数的 Invoker 的权重进行对比，
        // 以检测是否所有具有相同最小活跃数的 Invoker 的权重均相等
        int firstWeight = 0;
        boolean sameWeight = true;

        // 遍历 invokers 列表
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // 获取 Invoker 对应的活跃数
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // 获取权重 - ⭐️
            int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), Constants.WEIGHT_KEY, Constants.DEFAULT_WEIGHT);
            // 发现更小的活跃数，重新开始
            if (leastActive == -1 || active < leastActive) {
                // 使用当前活跃数 active 更新最小活跃数 leastActive
                leastActive = active;
                // 更新 leastCount 为 1
                leastCount = 1;
                // 记录当前下标值到 leastIndexs 中
                leastIndexs[0] = i;
                totalWeight = weight;
                firstWeight = weight;
                sameWeight = true;

            // 当前 Invoker 的活跃数 active 与最小活跃数 leastActive 相同 
            } else if (active == leastActive) {
                // 在 leastIndexs 中记录下当前 Invoker 在 invokers 集合中的下标
                leastIndexs[leastCount++] = i;
                // 累加权重
                totalWeight += weight;
                // 检测当前 Invoker 的权重与 firstWeight 是否相等，
                // 不相等则将 sameWeight 置为 false
                if (sameWeight && i > 0
                    && weight != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        
        // 当只有一个 Invoker 具有最小活跃数，此时直接返回该 Invoker 即可
        if (leastCount == 1) {
            return invokers.get(leastIndexs[0]);
        }

        // 有多个 Invoker 具有相同的最小活跃数，但他们的权重不同
        if (!sameWeight && totalWeight > 0) {
            // 随机获取一个 [0, totalWeight) 之间的数字
            int offsetWeight = random.nextInt(totalWeight);
            // 循环让随机数减去具有最小活跃数的 Invoker 的权重值，
            // 当 offset 小于等于0时，返回相应的 Invoker
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexs[i];
                // 获取权重值，并让随机数减去权重值 - ⭐️
                offsetWeight -= getWeight(invokers.get(leastIndex), invocation);
                if (offsetWeight <= 0)
                    return invokers.get(leastIndex);
            }
        }
        // 如果权重相同或权重为0时，随机返回一个 Invoker
        return invokers.get(leastIndexs[random.nextInt(leastCount)]);
    }
}
```

如上，为了帮助大家理解代码，我在上面的代码中写了大量的注释。下面简单总结一下以上代码所做的事情，如下：

1. 遍历 invokers 列表，寻找活跃数最小的 Invoker
2. 如果有多个 Invoker 具有相同的最小活跃数，此时记录下这些 Invoker 在 invokers 集合中的下标，以及累加它们的权重，比较它们之间的权重值是否相等
3. 如果只有一个 Invoker 具有最小的活跃数，此时直接返回该 Invoker 即可
4. 如果有多个 Invoker 具有最小活跃数，且它们的权重不相等，此时处理方式和 RandomLoadBalance 一致
5. 如果有多个 Invoker 具有最小活跃数，但它们的权重相等，此时随机返回一个即可

以上就是 LeastActiveLoadBalance 大致的实现逻辑，大家在阅读的源码的过程中要注意区分活跃数与权重这两个概念，不要混为一谈。

### 2.3 ConsistentHashLoadBalance

一致性 hash 算法由麻省理工学院的 Karger 及其合作者于1997年提供出的，算法提出之初是用于大规模缓存系统的负载均衡。

它的工作过程是这样的，首先根据 ip 获取其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 232 - 1]  的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash  值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash  值的缓存节点即可。

```java
public class ConsistentHashLoadBalance extends AbstractLoadBalance {

    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = 
        new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation);
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;

        // 获取 invokers 原始的 hashcode
        int identityHashCode = System.identityHashCode(invokers);
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        // 如果 invokers 是一个新的 List 对象，意味着服务提供者数量发生了变化，可能新增也可能减少了。
        // 此时 selector.identityHashCode != identityHashCode 条件成立
        if (selector == null || selector.identityHashCode != identityHashCode) {
            // 创建新的 ConsistentHashSelector
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }

        // 调用 ConsistentHashSelector 的 select 方法选择 Invoker
        return selector.select(invocation);
    }
```

如上，doSelect 方法主要做了一些前置工作，比如检测 invokers 列表是不是变动过，以及创建  ConsistentHashSelector。这些工作做完后，接下来开始调用 select 方法执行负载均衡逻辑。在分析 select  方法之前，我们先来看一下一致性 hash 选择器 ConsistentHashSelector 的初始化过程，如下：

```java
private static final class ConsistentHashSelector<T> {

    // 使用 TreeMap 存储 Invoker 虚拟节点
    private final TreeMap<Long, Invoker<T>> virtualInvokers;

    private final int replicaNumber;

    private final int identityHashCode;

    private final int[] argumentIndex;

    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
        this.identityHashCode = identityHashCode;
        URL url = invokers.get(0).getUrl();
        // 获取虚拟节点数，默认为160
        this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
        // 获取参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算
        String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) {
            argumentIndex[i] = Integer.parseInt(index[i]);
        }
        for (Invoker<T> invoker : invokers) {
            String address = invoker.getUrl().getAddress();
            for (int i = 0; i < replicaNumber / 4; i++) {
                // 对 address + i 进行 md5 运算，得到一个长度为16的字节数组
                byte[] digest = md5(address + i);
                // 对 digest 部分字节进行4次 hash 运算，得到四个不同的 long 型正整数
                for (int h = 0; h < 4; h++) {
                    // h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
                    // h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
                    // h = 2, h = 3 时过程同上
                    long m = hash(digest, h);
                    // 将 hash 到 invoker 的映射关系存储到 virtualInvokers 中，
                    // virtualInvokers 中的元素要有序，因此选用 TreeMap 作为存储结构
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }
}
```

ConsistentHashSelector 的构造方法执行了一系列的初始化逻辑，比如从配置中获取虚拟节点数以及参与 hash  计算的参数下标，默认情况下只使用第一个参数进行 hash。需要特别说明的是，ConsistentHashLoadBalance  的负载均衡逻辑只受参数值影响，具有相同参数值的请求将会被分配给同一个服务提供者。ConsistentHashLoadBalance 不 care 权重，因此使用时需要注意一下。

在获取虚拟节点数和参数下标配置后，接下来要做的事情是计算虚拟节点 hash 值，并将虚拟节点存储到 TreeMap 中。到此，ConsistentHashSelector 初始化工作就完成了。接下来，我们再来看看 select 方法的逻辑。

```java
public Invoker<T> select(Invocation invocation) {
    // 将参数转为 key
    String key = toKey(invocation.getArguments());
    // 对参数 key 进行 md5 运算
    byte[] digest = md5(key);
    // 取 digest 数组的前四个字节进行 hash 运算，再将 hash 值传给 selectForKey 方法，
    // 寻找合适的 Invoker
    return selectForKey(hash(digest, 0));
}

private Invoker<T> selectForKey(long hash) {
    // 到 TreeMap 中查找第一个节点值大于或等于当前 hash 的 Invoker
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
    // 如果 hash 大于 Invoker 在圆环上最大的位置，此时 entry = null，
    // 需要将 TreeMap 的头结点赋值给 entry
    if (entry == null) {
        entry = virtualInvokers.firstEntry();
    }

    // 返回 Invoker
    return entry.getValue();
}
```

如上，选择的过程比较简单了。首先是对参数进行 md5 以及 hash 运算，得到一个 hash 值。然后再拿这个值到 TreeMap 中查找目标 Invoker 即可。

到此关于 ConsistentHashLoadBalance 就分析完了。在阅读 ConsistentHashLoadBalance 之前，大家一定要先补充背景知识。否者即使这里只有一百多行代码，也很难看懂。好了，本节先分析到这。

### 2.4 RoundRobinLoadBalance

在详细分析源码前，我们还是先来了解一下什么是加权轮询。

这里从最简单的轮询开始讲起，所谓轮询就是将请求轮流分配给一组服务器。举个例子，我们有三台服务器 A、B、C。我们将第一个请求分配给服务器 A，第二个请求分配给服务器 B，第三个请求分配给服务器 C，第四个请求再次分配给服务器 A。这个过程就叫做轮询。

轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。

显然，现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然是不合理的。

因此，这个时候我们需要加权轮询算法，对轮询过程进行干预，使得性能好的服务器可以得到更多的请求，性能差的得到的少一些。每台服务器能够得到的请求数比例，接近或等于他们的权重比。

比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将获取到其中的5次请求，服务器 B 获取到其中的2次请求，服务器 C 则获取到其中的1次请求。

以上就是加权轮询的算法思想，搞懂了这个思想，接下来我们就可以分析源码了。我们先来看一下 2.6.4 版本的 RoundRobinLoadBalance。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "roundrobin";

    private final ConcurrentMap<String, AtomicPositiveInteger> sequences = 
        new ConcurrentHashMap<String, AtomicPositiveInteger>();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // key = 全限定类名 + "." + 方法名，比如 com.xxx.DemoService.sayHello
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        int length = invokers.size();
        // 最大权重
        int maxWeight = 0;
        // 最小权重
        int minWeight = Integer.MAX_VALUE;
        final LinkedHashMap<Invoker<T>, IntegerWrapper> invokerToWeightMap = new LinkedHashMap<Invoker<T>, IntegerWrapper>();
        // 权重总和
        int weightSum = 0;

        // 下面这个循环主要用于查找最大和最小权重，计算权重总和等
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // 获取最大和最小权重
            maxWeight = Math.max(maxWeight, weight);
            minWeight = Math.min(minWeight, weight);
            if (weight > 0) {
                // 将 weight 封装到 IntegerWrapper 中
                invokerToWeightMap.put(invokers.get(i), new IntegerWrapper(weight));
                // 累加权重
                weightSum += weight;
            }
        }

        // 查找 key 对应的对应 AtomicPositiveInteger 实例，为空则创建。
        // 这里可以把 AtomicPositiveInteger 看成一个黑盒，大家只要知道
        // AtomicPositiveInteger 用于记录服务的调用编号即可。至于细节，
        // 大家如果感兴趣，可以自行分析
        AtomicPositiveInteger sequence = sequences.get(key);
        if (sequence == null) {
            sequences.putIfAbsent(key, new AtomicPositiveInteger());
            sequence = sequences.get(key);
        }

        // 获取当前的调用编号
        int currentSequence = sequence.getAndIncrement();
        // 如果 最小权重 < 最大权重，表明服务提供者之间的权重是不相等的
        if (maxWeight > 0 && minWeight < maxWeight) {
            // 使用调用编号对权重总和进行取余操作
            int mod = currentSequence % weightSum;
            // 进行 maxWeight 次遍历
            for (int i = 0; i < maxWeight; i++) {
                // 遍历 invokerToWeightMap
                for (Map.Entry<Invoker<T>, IntegerWrapper> each : invokerToWeightMap.entrySet()) {
                    // 获取 Invoker
                    final Invoker<T> k = each.getKey();
                    // 获取权重包装类 IntegerWrapper
                    final IntegerWrapper v = each.getValue();
                    
                    // 如果 mod = 0，且权重大于0，此时返回相应的 Invoker
                    if (mod == 0 && v.getValue() > 0) {
                        return k;
                    }
                    
                    // mod != 0，且权重大于0，此时对权重和 mod 分别进行自减操作
                    if (v.getValue() > 0) {
                        v.decrement();
                        mod--;
                    }
                }
            }
        }
        
        // 服务提供者之间的权重相等，此时通过轮询选择 Invoker
        return invokers.get(currentSequence % length);
    }

    // IntegerWrapper 是一个 int 包装类，主要包含了一个自减方法。
    // 与 Integer 不同，Integer 是不可变类，而 IntegerWrapper 是可变类
    private static final class IntegerWrapper {
        private int value;

        public void decrement() {
            this.value--;
        }
        
        // 省略部分代码
    }
}
```



https://segmentfault.com/a/1190000023408304?utm_source=tag-newest