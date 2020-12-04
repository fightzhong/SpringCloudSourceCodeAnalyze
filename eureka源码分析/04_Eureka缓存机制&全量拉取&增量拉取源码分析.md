一、GuavaCache的简单使用
```
Guava是谷歌开源的一个工具库, 提供了许多强大而有用的功能, 如集合、缓冲池、并发库、字符串处理等, 说
实话这个库我用的也不多, 在学生时代写的第一个项目就用过其缓存功能, 对其理解也不深刻, 然而在Eureka注
册表的多级缓存功能中就用Guava Cache来实现了其中的二级缓存, 不过guava的使用相当简单, 我们这里来简单
的说下关于Cache这一部分的使用吧
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
                                .maximumSize(10000)
                                .expireAfterWrite(10, TimeUnit.MINUTES)
                                .removalListener(MY_LISTENER)
                                .build( new CacheLoader<Key, Graph>() {
                                        public Graph load(Key key) throws AnyException {
                                        return createExpensiveGraph(key);
                                        }
                                });

以上这个对GuavaCache的使用例子来源于Guava中的CacheBuilder的javadoc文档, LoadingCache就是最终的
缓存对象, 通过CacheBuilder建造者模式进行建造, maximumSize这一行代码表示在LoadingCache中最多能有
10000个缓存对象, expireAfterWrite这一行代码表示一个缓存对象的存活时间为从被创建或者被更新开始往后
数10分钟, 即10分钟没被更新就会从缓存中移除, removalListener表示添加一个监听器, 当缓存被移除的时候
会触发该监听器的调用, build操作提供一个CacheLoader, 作用是, 当从缓存中没有取到需要的值的时候, 就
会触发这个CacheLoader的load方法调用, 该方法的返回值将会被放入到缓存中, 从而使得下次用相同的key取
的时候, 能够取到该缓存的值

graphs.get( key )方法的调用, 会先从缓存中拿取缓存值, 如果缓存中没有存在, 就触发CacheLoader中load
方法的调用, 从而得到一个值, 这个值会被放入到graphs中, 第二次调用graphs.get( key )方法的时候就能
从缓存中取到值了

以上就是GuavaCache的简单使用
```


二、Eureka的多级缓存原理分析
```java
在之前的文章中, 我们对Eureka的服务注册、服务下线等等功能进行了详细分析, 在这些分析中, 我们得知了
在Eureka中存储注册表的结构是一个双层的Map:
    ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry;
这就是多级缓存中的第一层缓存, 或者说真正存储完整数据的地方

当我们注册的服务越来越多的时候, 客户端服务就会每隔一段时间拉取注册表的数据(默认是30秒拉取一次), 而
注册操作、下线操作一般是比较少的(谁没事会动不动的把线上跑着的服务关闭或者重启呀), 为了尽可能的提高
更新操作和读取操作之间的并发, Eureka提供了第二层缓存, 这个缓存就是基于GuavaCache实现的, 我们先来
看看第二层缓存的创建吧(省略了一些代码):
public class ResponseCacheImpl {
    private final LoadingCache<Key, Value> readWriteCacheMap;
    
    ResponseCacheImpl () {
        this.readWriteCacheMap = CacheBuilder.newBuilder()
            .initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
            .expireAfterWrite(
                serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
            .build(new CacheLoader<Key, Value>() {
                @Override
                public Value load(Key key) throws Exception {
                    return generatePayload(key);
                }
            });
    }
}
第二层缓存就是这个readWriteCacheMap, 可以看到, 就是我们上面说的GuavaCache的构建过程, 缓存的大小
基于eureka.server.initial-capacity-of-response-cache这个配置, 默认为1000个, 缓存的有效时间为
eureka.server.response-cache-auto-expiration-in-seconds这个配置, 默认为180秒, 当缓存中不存在
的时候, 就会调用generatePayload方法来生成数据, 第二层缓存中, 缓存的键是Key这个类型的, 缓存的值是
Value这个类型的

再往后, Eureka又提供了第三层缓存, 第三层缓存其实非常简单, 就是一个基于上面的Key类型和Value类型组
成的Map:
    ConcurrentMap<Key, Value> readOnlyCacheMap = new ConcurrentHashMap<Key, Value>()

这就是三层缓存的基本结构了, 那么三层缓存中如何进行同步的呢? 通常情况下, 当服务发现的时候, 会先从
readOnlyCacheMap中取得全量或者增量数据, 如果readOnlyCacheMap中没有, 则从readWriteCacheMap中获
取值, 如果在readWriteCacheMap中也没有缓存数据, 那么就会触发CacheLoader的load方法, 开始从真正的
注册表中拉取数据(增量拉取的原理后面我们再分析, 这里是全量拉取的情况), 进而放入到readWriteCacheMap
中

而readOnlyCacheMap会利用一个定时器, 每隔30秒从readWriteCacheMap中进行数据的同步 

总结:
    <1> 三级缓存的结构分别为:
        ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry;
        LoadingCache<Key, Value> readWriteCacheMap;
        ConcurrentMap<Key, Value> readOnlyCacheMap;
    <2> readOnlyCacheMap通过定时器每隔30秒从readWriteCacheMap中同步数据
    <3> readWriteCacheMap中查找缓存数据没找到的情况下, 会利用CacheLoader从registry中同步数据
        (全量拉取或者固定拉取一个服务的数据的情况下)
    <4> 增量拉取的原理后面我们再单独分析
```


三、缓存中键值对数据结构分析
```java
Eureka提供了服务发现的功能, 而缓存都是用于这个服务发现的, 服务发现有好几种情况, 在此举三个例子, 第
一种是全量拉取, 就是拉取整个注册表的数据, 第二种是增量拉取, 仅仅拉取一段时间内更新过的数据, 第三种
是仅仅拉取一类服务的注册信息, 比如购物车服务, 而缓存的Key就是根据这些来进行计算的, 下面我们来看看
全量拉取的Key的定义吧:
    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS,
            Key.KeyType.JSON, CurrentRequestVersion.get(), 
            EurekaAccept.fromString(eurekaAccept), regions
    );
其实这几个参数都是固定的, 没有可变的参数, 再来看看增量拉取的Key的定义: 
    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS_DELTA,
            Key.KeyType.JSON, CurrentRequestVersion.get(), 
            EurekaAccept.fromString(eurekaAccept), regions
    );

最后我们来看看Key的内部结构吧:
public class Key {
     public Key(EntityType entityType, String entityName, KeyType type, Version v, 
                EurekaAccept eurekaAccept, String[] regions) {
        this.regions = regions;
        this.entityType = entityType;
        this.entityName = entityName;
        this.requestType = type;
        this.requestVersion = v;
        this.eurekaAccept = eurekaAccept;
        hashKey = this.entityType + this.entityName + 
                (null != this.regions ? Arrays.toString(this.regions) : "")
                + requestType.name() + requestVersion.name() + this.eurekaAccept.name();
    }

    public int hashCode() {
        String hashKey = getHashKey();
        return hashKey.hashCode();
    }

    public boolean equals(Object other) {
        if (other instanceof Key) {
            return getHashKey().equals(((Key) other).getHashKey());
        } else {
            return false;
        }
    }
}

其实很简单, 就是存储了几个变量的值而已, 由于Key是作为Map中的键的, 所以需要重写hashCode和equals
方法, 而这两个方法的计算就是基于这几个成员变量的, 所以我们可以得出一个结论, 全量拉取和增量拉取的
Key是固定的

再来看看Value的结构:
    public class Value {
        private final String payload;
        private byte[] gzipped;
    }

其实很简单, 由于我们保存的是注册表的信息, 即payload这个字符串其实就是注册表信息的JSON表现形式(通
常情况下, 当然还有可能是XML表现形式, 这是可配置的), gzipped这个byte数组其实就是payload这个JSON
经过gzip压缩之后的字节数据而已
```


四、缓存值的构造过程

1、入口--generatePayload方法
```java
在上面, 我们知道了, 当从二级缓存readWriteCacheMap中没有获取到注册信息的时候, 就会调用CacheLoader
的load方法来构造Value, 我们来看看这个Value的构造过程吧(简略代码):
private Value generatePayload(Key key) {
    String payload;
    switch (key.getEntityType()) {
        case Application:
            if (ALL_APPS.equals(key.getName())) {
                payload = getPayLoad(key, registry.getApplications());
            } else if (ALL_APPS_DELTA.equals(key.getName())) {
                payload = getPayLoad(key, registry.getApplicationDeltas());
            } else {
                payload = getPayLoad(key, registry.getApplication(key.getName()));
            }
            break;
        case VIP:
        case SVIP:
            ..............................
    }

    return new Value(payload);
}

分析:
    关于VIP和SVIP这两个情况我们不进行分析, 大家有兴趣可以去了解下, 这里简单的提一下, 客户端现在从
    EurekaServer拉取数据的时候, 默认拉取的是全部的注册表数据, 但是Eureka有提供配置, 即VIP和SVIP
    这两个配置, 用于指示只拉取指定的一类服务的注册信息, 比如我可以配置VIP为购物车服务, 那么作为客
    户端的我, 每次从EurekaServer中拉取的注册表信息就仅仅会是购物车服务的注册表信息了

    接下来我们回到正题, 可以看到上面的代码中, 对于全量拉取和增量拉取会分别调用registry的
    getApplications方法和getApplicationDeltas方法, 最后调用getPayLoad方法来生成payload这个JSON
    数据了, 最后new了一个Value并返回, 所以我们要分别来了解下这三个方法的源码
```


2、getApplications方法(全量拉取)
```java
当拉取数据为全量拉取的时候, 就会调用该方法, 其实非常简单, 就是将registry这个真正存储注册信息的Map
中的所有注册表信息进行遍历并放到Applications对象中, 伪代码如下:
    Applications apps = new Applications();
    for ( registry.entrySet() ) {
        // 遍历一个个的注册表信息, 放入倒apps中
    }
    apps.setAppsHashCode(apps.getReconcileHashCode());
注意最后一行代码, Applications中最终会设置hashcode为所有注册表信息计算出来的hashcode值
```

3、getApplicationDeltas(增量拉取)
```java
终于说到了增量拉取了, 在之前的分析中, 我们提到了一个recentlyChangedQueue的队列, 在服务注册、服务
下线的时候, 就会往这个队列中放入对应的注册表对象, 而增量拉取的原理很简单, 就是遍历这个队列, 将里面
存储的信息一个个取出来返回给客户端就好了, 代码如下:
    public Applications getApplicationDeltas() {
        Applications apps = new Applications();

        Iterator<RecentlyChangedItem> iter = this.recentlyChangedQueue.iterator();
         while (iter.hasNext()) {
            Lease<InstanceInfo> lease = iter.next().getLeaseInfo();
            InstanceInfo instanceInfo = lease.getHolder();
            Application app = applicationInstancesMap.get(instanceInfo.getAppName());
            if (app == null) {
                app = new Application(instanceInfo.getAppName());
                applicationInstancesMap.put(instanceInfo.getAppName(), app);
                apps.addApplication(app);
            }
            app.addInstance(new InstanceInfo(decorateInstanceInfo(lease)));
        }

        apps.setAppsHashCode(allApps.getReconcileHashCode());
    }

很清晰, 就是遍历recentlyChangedQueue中的item, 然后放入到apps中, 这里也要注意最后一行代码, apps
中虽然放的是队列中存储的增量信息, 但是apps的hashCode却是registry中完整的注册信息计算得出的
hashcode!!!
```

4、getPayLoad生成Value
```java
通过上面两节, 我们可以得到, 不管是全量拉取还是增量拉取对应的方法, 最终都会获得一个Applications注
册表存储对象, 如果是前者, 则存放的是整个EurekaServer中的注册表信息, 如果是后者, 则存放的是增量的
注册表信息, 但是, Applications对象中的hashCode却是通过EurekaServer中所有的注册表信息得出来的!!!

private String getPayLoad(Key key, Applications apps) {
    EncoderWrapper encoderWrapper = serverCodecs.getEncoder(key.getType(), key.getEurekaAccept());
    return encoderWrapper.encode(apps);
}
getPayLoad方法其实很简单, 就是对Applications进行编码而已, 比如编码成一个JSON对象
```


五、全量拉取接口源码分析(增量拉取类似)
```java
接下来我们终于可以来看全量拉取接口的源码了, 在ApplicationsResource这个类中:

public Response getContainers(@PathParam("version") String version,
                    @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                    @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
                    @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
                    @Context UriInfo uriInfo,
                    @Nullable @QueryParam("regions") String regionsStr) {
    -------------------------------------第一部分-------------------------------------
    CurrentRequestVersion.set(Version.toEnum(version));
    KeyType keyType = Key.KeyType.JSON;
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }

    -------------------------------------第二部分-------------------------------------
    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS,
            keyType, CurrentRequestVersion.get(), 
            EurekaAccept.fromString(eurekaAccept), regions
    );


    -------------------------------------第三部分-------------------------------------
    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        response = Response.ok(responseCache.getGZIP(cacheKey))
                .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
                .header(HEADER_CONTENT_TYPE, returnMediaType)
                .build();
    } else {
        response = Response.ok(responseCache.get(cacheKey))
                .build();
    }
    return response;
}

分析:
    getContainers就是全量拉取的接口了, 在上面, 我分成了三部分
    第一部分, 对构建缓存Key的参数进行赋值
    第二部分, 构建缓存Key
    第三部分, 判断是返回gzip压缩后的注册表信息还是直接将非压缩后的信息进行返回, 如果是前者, 就是
             返回Value中的gzipped这个数组, 如果是后者, 就是返回Value中的payload

我们来看看responseCache.get(cacheKey)这段获取压缩前数据的源码吧(getGZIP方法也类似):
String get(final Key key, boolean useReadOnlyCache) {
    Value payload = getValue(key, useReadOnlyCache);
    return payload.getPayload();
}

Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    if (useReadOnlyCache) {
        final Value currentPayload = readOnlyCacheMap.get(key);
        if (currentPayload != null) {
            payload = currentPayload;
        } else {
            payload = readWriteCacheMap.get(key);
            readOnlyCacheMap.put(key, payload);
        }
    } else {
        payload = readWriteCacheMap.get(key);
    }
    
    return payload;
}

分析:
    可以看到, 最终调用到了getValue方法, 如果不允许使用ReadOnly这一层缓存, 就直接从GuavaCache构成
    的readWriteCacheMap这层缓存中取了, 如果允许, 则会先从readOnlyCacheMap中取, 取不到才从
    readWriteCacheMap中取, 如果readWriteCacheMap中取不到, 那么就会触发上面我们分析的
    generatePayload方法的调用了(CacheLoader的load方法)
```

六、小小的总结
```
从GuavaCache出发, 我们了解到了二级缓存readWriteCacheMap的使用情况, 然后我们引出了Eureka中的三级
缓存的结构:
    <1> ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry;
    <2> LoadingCache<Key, Value> readWriteCacheMap;
    <3> ConcurrentMap<Key, Value> readOnlyCacheMap;

再之后, 分析了缓存中Key、Value两者的结构, 前者通过一些参数构成一个hashcode来确定唯一, 后者有两个
成员变量分别用来保存压缩前的数据和压缩后的数据

了解了缓存中键值的数据结构后, 我们开始对缓存值的构造进行了分析, 在缓存值的构造中, 对全量拉取时数据
的构造和增量拉取时数据的构造源码进行了分析, 前者取的整个注册表信息, 后者取的是recentlyChangedQueue
这个队列中的数据, 而这个队列在进行服务注册、服务下线等等一系列操作的时候都会被写入当前触发该操作的
服务的数据

最后我们分析了全量拉取接口的源码, 先是构造Key对象, 然后从缓存中取, 先从readOnlyCacheMap中取, 取不
到或者说不允许从readOnlyCacheMap中取, 就会从readWriteCacheMap中取, 如果还取不到, 就触发
GuavaCache的CacheLoader中的load方法, 来加载缓存了, 即上一段话中的数据构造过程
```

七、扩展
```
在了解了EurekaServer中的缓存机制、全量拉取和增量拉取的相关源码后, 我们还余留了知识点(客户端拉取数
据的源码分析, 全量拉取和增量拉取), 这一个知识点就不再进行文章的展开分析了, 否则文章会很长, 这里提出
主要理论原理, 大家有兴趣可以去研究研究, 有了这篇文章的铺垫后, 去研究这个知识点就会轻松多了

在我们上面的分析中, 会对Applications中的hashcode进行设置, 当客户端调用增量拉取接口的时, 取的是服
务端中recentlyChangedQueue这个队列中的数据, 然而这个队列会在一个间隔时间为30秒的定时器中进行更
新, 更新的操作是, 将队列中所有的注册表信息中, lastUpdateTime即最近更新时间与当前时间进行比较, 如
果超过了三分钟, 则将这个注册表从队列中移除, 而这个hashcode的作用在于, 当客户端拉取到增量信息并整合
到自己的本地缓存后, 将本地缓存中的所有注册表进行hashCode计算, 此时与EurekaServer返回的
Applications中的hashCode进行比较, 如果两者不一致, 说明数据不一致了, 此时会触发全量拉取

其次, 我们上面提到了VIP这个概念, 表示客户端仅仅拉取指定的注册表信息, 如果有对这个进行配置, 那么客
户端就不会触发增量拉取的操作, 而是一直采用全量拉取来拉取数据
```
