一、引入
```
这一个系列是针对于Eureka源码分析的系列, 现在大部分公司都会开始将自身的业务进行微服务化, 其中Java这边微服务大部分都是采用
SpringCloud一套, 深入的理解各个微服务组件的原理, 能够使得我们在进行微服务拆分的时候就会有更多的思考方向, 对每个组件的使
用也会更加的得心应手, 以及对这些组件有优化思路, Eureka作为注册中心, 核心功能就是服务注册、服务发现、服务续约、以及集群模
式下的同步机制, 这个系列就会对这几块内容进行深入的分析, 如果有看过笔者之前的Netty系列以及SpringMvc系列的源码分析的同学
可能对笔者写文章的结构会有所了解, 那就是从一个个小型的组件分析开始, 最终再分析一个功能的全部流程, 比如在Netty中, 我没有
一上来就分析服务端启动的源码, 而是先开始对启动源码涉及到的组件进行一个个分析, 这样的好处是, 在了解了这些基础组件的情况下
再去读依托于这些组件的源码就会非常的顺畅

分析Eureka源码也是类似, 我不会一开始就从Eureka在SpringCloud中的stater自动配置类进行分析, 而是先分析基础组件、基础功能,
本篇文章分析的就是Eureka自我保护机制的源码

同时, 这里需要注意的一点是, 笔者分析源码时, 在文章中贴出来的代码通常是剔除了异常处理、日志输出、以及一些对理解主线核心原
理没有帮助的代码, 换句话说, 如果要读懂某一个功能, 这些代码完全是可以不用看的, 如果有看过之前笔者写的文章, 那么就应该会熟
悉这一点了
```

二、测量器 / 计数器
```java
public class MeasuredRate {
    private final AtomicLong lastBucket = new AtomicLong(0);
    private final AtomicLong currentBucket = new AtomicLong(0);
    private final long sampleInterval;
    private final Timer timer;

    public MeasuredRate(long sampleInterval) {
        this.sampleInterval = sampleInterval;
        this.timer = new Timer("Eureka-MeasureRateTimer", true);
    }

    public synchronized void start() {
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                lastBucket.set(currentBucket.getAndSet(0));
            }
        }, sampleInterval, sampleInterval);
    }

    public void increment() {
        currentBucket.incrementAndGet();
    }

    public long getCount() {
        return lastBucket.get();
    }
}

其实这个MeasuredRate计数器很简单, 就是有一个定时器每隔sampleInterval会执行一次而已, 执行的任务其实很简单, 就是将
lastBucket设置为currentBucket的值, 而将currentBucket重新设置为0, currentBucket其实就是一个计数器而已, 可以通过
increment方法对其进加法操作, 而getCount方法则获取的是上一个执行周期的值

举个例子:  
    MeasuredRate measuredRate = new MeasuredRate( 60 * 1000 );
    measuredRate.start();

上面这段代码执行完毕后, 每分钟就会将currentBucket的值赋予给lastBucket, 而currentBucket被置为0, 在这一分钟内可以执行多
次increment操作去更新currentBucket的值
```

三、自我保护机制源码分析

3.1、什么是Eureka的自我保护机制
```
EurekaServer保存了所有注册的实例, EurekaClient实例通过心跳机制来保持与EurekaServer联系, 告知EurekaServer自身是存活状
态, 心跳机制其实就是每隔一段时间发送一个请求到EurekaServer而已, 那么就会出现一个问题, 当客户端本身状态正常的情况下, 由于
EurekaServer网络的不流畅, 会导致EurekaServer不能及时的接收到客户端发来的心跳包, 这个时候EurekaServer就可能会将该
客户端定义为非存活状态, 想象一个场景, 当实例很多的时候, EurekaServer因为一些特殊原因, 与部分客户端之间的网络出现了问题,
此时刚好执行了剔除实例的功能, 就可能会认为这些客户端都不是存活的了, 于是就将他们从内存中剔除掉, 而与EurekaServer还处于正
常连接的客户端, 此时就会获取不到这些被剔除的服务了

为了防止这种情况的发生, Eureka自我保护机制就出来了, 根据较为官方的描述: 在运行期间EurekaServer会去统计心跳失败比例在15
分钟之内是否低于85%，如果低于85%，EurekaServer会将剩下的实例保护起来, 让这些实例不会过期

通俗的话来讲, 就是15分钟内服务剔除的比例达到了注册的15%, 那么EurekaServer就开启了自我保护机制, 不再将服务进行剔除
```


3.2、涉及到的配置项分析
```
我们先来聊一下几个关于服务续约的配置吧............

第一个是eureka.instance.lease-renewal-interval-in-seconds, 这是客户端服务续约的间隔, 参数是数值型, 通俗的话来说, 就
是客户端每隔该配置的时间就会发送一个请求告诉Eureka的服务端自己还活着

第二个是eureka.instance.lease-expiration-duration-in-seconds, 这个参数也是数值型, 通俗的话来说, 就说客户端告诉
EurekaServer, 如果超过了该配置的时间, 我还没有再一次发送心跳到服务端的话, 那么就把我剔除掉吧, 通常会应用在, 客户端非法
关闭, 比如强杀Eureka客户端, 这样Eureka客户端就不会进行心跳包发送了, 并且也没有主动发送下线请求, 从而成为了EurekaServer
中的非法服务

第三个是eureka.server.enable-self-preservation, 如果为true, 则Eureka会开启自我保护机制, 如果为false, 则不会开启自我
保护机制, 默认是true, 即开启了自我保护机制

第四个是eureka.server.renewal-percent-threshold, 这是服务续约的阈值, 默认值是0.85, 即85%, 结合2.1节中对自我保护机制
的描述, 其实就是指15分钟内EurekaServer没有收到心跳数大于等于该配置规定的比例的话, EurekaServer就会开启自我保护机制

第五个是eureka.server.expected-client-renewal-interval-seconds, 这个配置大家应该几乎是没用过的, 这个配置跟第一个配置
有点类似, 指的是服务端期望客户端服务续约的间隔, 通俗的话来说, 就是EurekaServer期望客户端每隔该配置所定义的时间发送一个心
跳包到EurekaServer
```


3.3、源码分析
```java
在真正进行源码分析之前, 我们再来聊几个变量:

expectedNumberOfClientsSendingRenews: EurekaServer期望收到的服务续约的客户端个数, 其实很简单, 就是初始值为0, 只要有
一个服务注册上来, 那么就加1, 有一个服务主动下线, 那么就减1, 注意了, 是主动下线, 而EurekaServer主动剔除的是不会更改这个
值的

numberOfRenewsPerMinThreshold: 每分钟续约的阈值, 同时也是自我保护机制是否启动的标志,  如果每分钟续约的客户端个数(即每
分钟发送心跳的客户端数)大于了这个值, 就不会开启自我保护机制, 反之, 则启动自我保护机制, 不再剔除过期的实例

在了解了以上五个配置加两个参数后, 我们再来看自我保护机制的源码就会轻松多了:
public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry{
    public boolean isLeaseExpirationEnabled() {
        if (!isSelfPreservationModeEnabled()) {
            return true;
        }
        return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
    }
}

public abstract class AbstractInstanceRegistry {
    private final MeasuredRate renewsLastMin;

    protected volatile int numberOfRenewsPerMinThreshold;

    protected AbstractInstanceRegistry (EurekaServerConfig serverConfig, 
                                        EurekaClientConfig clientConfig, ServerCodecs serverCodecs) {
        ..........一系列的赋值操作..........
        this.renewsLastMin = new MeasuredRate(1000 * 60 * 1);
        ..........一系列的赋值操作..........
    }

    public long getNumOfRenewsInLastMin() {
        return renewsLastMin.getCount();
    }

    protected void updateRenewsPerMinThreshold() {
        this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
                * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
                * serverConfig.getRenewalPercentThreshold());
    }
}

首先我们要了解, Eureka的自我保护机制是作用于服务剔除时的, 自我保护机制和服务剔除是息息相关的, 服务剔除的源码我们先不用
理会, 之后会详细介绍(EurekaServer会有一个地方保存注册的实例, 如果发生服务下线或者服务剔除, 就从这个地方将这个服务删除掉)

上面几句代码, 就是Eureka自我保护机制的源码了, 其实很简单, 我们先来看看PeerAwareInstanceRegistryImpl中的代码吧, 是一个
判断, isLeaseExpirationEnabled方法, 见名思意, 就说是否允许节点续约过期, 其实这个方法是在EurekaServer剔除节点前进行调
用的, EurekaServer会定时的扫描所有注册到其上面的实例, 判断一个实例是否过期就是根据上面我们说的五个参数中的第二个即
eureka.instance.lease-expiration-duration-in-seconds来判断的

一旦过期, 那么就会从EurekaServer中剔除, 而isLeaseExpirationEnabled方法就是判断是否要进行剔除的操作, 结合自我保护机制
进行理解, 一旦触发了自我保护机制, 就不会进行节点的剔除操作了, 换句话说, 一旦该方法返回false, 说明不允许节点过期, 则不会
执行剔除操作, 所以在服务剔除之前会进行这个判断, 于是我们来看看该方法的逻辑吧

isLeaseExpirationEnabled方法逻辑分析:
    首先看到, 第一个判断, 如果自我保护模式没有被启用, 那么就返回true, 即表示会进行剔除操作, 其实也好理解, 都没有启用自我
    保护功能, 自然就不用考虑自我保护机制了, 那么该剔除过期节点就会进行剔除了, 判断是否启用自我保护功能, 用的配置是
    eureka.server.enable-self-preservation, 换句话说, 这个判断其实就是判断这个配置是true还是false

那么仅仅当自我保护功能开启的时候, 即eureka.server.enable-self-preservation为true的时候, 才会走后面的逻辑去判断此时是
否触发了自我保护机制

numberOfRenewsPerMinThreshold, EurekaServer期望每分钟收到的心跳值, 举个简单的例子, 我作为一个EurekaServer, 注册到我
这里的客户端有10个, 这些客户端每隔30秒发送一个心跳包给我, 那么每分钟我期望收到的心跳包是10 * 2即20个, 但是, 我要考虑到
网络存在不稳定的情况, 于是我降低要求, 只要85%的客户端能够发送心跳包给我就够了, 于是numberOfRenewsPerMinThreshold的值
就变成了20 * 0.85, 这就是numberOfRenewsPerMinThreshold字段的计算方式, 不过需要注意的是, 客户端每隔多长时间发送心跳包
是客户端可以自己配置的, 作为服务端我没法干预, 所以我采用eureka.server.expected-client-renewal-interval-seconds这个服
务端的配置来当作客户端发送心跳包的频率......

于是我们再来看updateRenewsPerMinThreshold方法就会清晰多了, 其实里面就是更新了numberOfRenewsPerMinThreshold值, 而这个
值的计算就是用以下几个配置计算处理的:
    <1> expectedNumberOfClientsSendingRenews
    <2> eureka.server.expected-client-renewal-interval-seconds
    <3> eureka.server.renewal-percent-threshold
计算的代码中还有一个除法操作, 60表示60秒的意思, 结合上面的描述, 应该很容易理解numberOfRenewsPerMinThreshold的计算方式
了, 那么updateRenewsPerMinThreshold方法什么时候会被调用呢???很简单, 计算的方式中只有一个值会在运行的时候被改变, 就是
expectedNumberOfClientsSendingRenews参数, 服务的上线会导致该参数加1, 服务的主动下线会导致该参数减1, 所以这个方法会在
这两个地方被调用(还有是初始化的时候, 这个我们可以暂时不用理会, 因为Eureka在初始化的时候会从其他集群节点中拉取注册的服务
信息, 拉取完后自然也要更新expectedNumberOfClientsSendingRenews值了), 这个时候我们再来理解
numberOfRenewsPerMinThreshold参数的意思就会清晰多了, EurekaServer期望每分钟收到的心跳包的个数...

此时我们再回到PeerAwareInstanceRegistryImpl类中的isLeaseExpirationEnabled方法, 最后有一段代码, 这段代码返回true表示
子我们保护机制没有触发, 允许节点过期, 从而被剔除, 返回false, 表示自我保护机制被触发, 不允许剔除节点, 而什么时候触发自我
保护机制呢???即每分钟发送心跳包的个数没有达到numberOfRenewsPerMinThreshold值!!!!

getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold ---> 不会触发自我保护机制
getNumOfRenewsInLastMin() <= numberOfRenewsPerMinThreshold ---> 会触发自我保护机制

public long getNumOfRenewsInLastMin() {
    return renewsLastMin.getCount();
}

而这个renewsLastMin就是一个计数器!!!即本篇文章一开始的时候说的那个计数器!!!这个计数器有一个increment方法, 这个方法会在
EurekaServer每收到一个心跳包的时候调用一次!!!

还是举个例子, EurekaServer上注册了100个节点, 每分钟期望收到的心跳数是100 * 0.85 * 2 即170个心跳包, renewsLastMin会记
录每分钟的心跳数量, 如果少于170个就会进入自我保护模式, 从而不再剔除节点信息
    
```


3.4、小小的总结
```
有一个计数器renewsLastMin, 这个计数器的lastBucket属性每分钟会保存这一分钟currentBucket经过多次increment操作后的值, 
然后currentBucket又被置为0, 从而开始下一个周期的统计

EurekaServer为了折中起见, 认为客户端应该按照自己的eureka.server.expected-client-renewal-interval-seconds配置的时间
间隔来发送心跳包, EurekaServer中每注册一个服务和每主动下线一个服务都会去更新自己的expectedNumberOfClientsSendingRenews
属性, 来保存自身当中实际的服务数量, 利用上面两个配置以及续约阈值eureka.server.renewal-percent-threshold配置, 
EurekaServer能够计算出每分钟应该接收多少个心跳包, 如果没有接收到这么多个心跳包, EurekaServer就会触发自我保护机制, 从而
不再剔除节点, 这个就是Eureka自我保护机制的原理

通过源码, 我们可以知道, Eureka在开启了自我保护机制后, 下面三个参数会影响EurekaServer对自身是否应该进入自我保护模式的判
断: 
    <1> eureka.instance.lease-renewal-interval-in-seconds
    <2> eureka.server.expected-client-renewal-interval-seconds
    <3> eureka.server.renewal-percent-threshold
所以当我们自己在进行EurekaServer的心跳优化的时候, 如果开启了自我保护机制, 一定要慎重这三个参数的配置!!!
```


3.5、自我保护机制的扩展
```java
上面我们也有说到, EurekaServer运行的过程中, numberOfRenewsPerMinThreshold这个字段的计算只有一个值会发生改变, 
即expectedNumberOfClientsSendingRenews

但是这个值发生改变的只会在服务注册和服务主动下线的时候, 也就是说, 在一段时间内, 服务没有发生注册和下线, 那么
numberOfRenewsPerMinThreshold这个字段的值就不会进行变更, 假如有100个服务, 因为网络原因有15个服务被剔除了, 此时触发了
自我保护机制, 由于numberOfRenewsPerMinThreshold这个字段一直没有变更, 会导致EurekaServer一直处于自我保护机制中

然而还有一个定时器也会更新expectedNumberOfClientsSendingRenews的值, 这个定时器通常作用于Eureka集群的模式下, 其作用是,
每隔15分钟, 执行一个任务, 源码如下(PeerAwareInstanceRegistryImpl的scheduleRenewalThresholdUpdateTask方法):
timer.schedule(new TimerTask() {
    @Override
    public void run() {
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        
        if ((count) > (serverConfig.getRenewalPercentThreshold() * expectedNumberOfClientsSendingRenews)
                || (!this.isSelfPreservationModeEnabled())) {
            this.expectedNumberOfClientsSendingRenews = count;
            updateRenewsPerMinThreshold();
        }
    }
}, serverConfig.getRenewalThresholdUpdateIntervalMs(),
serverConfig.getRenewalThresholdUpdateIntervalMs())

这段代码的作用在EurekaServer中有注释, 如下:
    计划定期更新续订阈值的任务, 更新阈值将用于确定更新是否由于网络分区而急剧下降, 并保护一次被剔除的实例过多

不过这一段代码一般在集群模式下才会有更新的效果, 第一行代码eurekaClient.getApplications()就说明了当前EurekaServer是被
当作客户端来对待的, eurekaClient.getApplications()其实就是从其它集群实例获取到的实例个数
```
