一、HttpRequestWrapper
```java
public class HttpRequestWrapper implements HttpRequest {
    private final HttpRequest request;
}

在第一篇文章中, 我们对RestTemplate源码进行了分析, 先来回顾下执行流程:
    利用ClientHttpRequestFactory的createRequest方法来创建一个ClientHttpRequest对象, 进而调用
    该对象的execute方法, 当存在拦截器并且拦截器中没有实现发送请求功能的时候, 代码的执行逻辑是这
    样的:
        <1> 创建InterceptingClientHttpRequestFactory并调用器createRequest方法, 该方法返回一个
            InterceptingClientHttpRequest对象
        <2> 调用InterceptingClientHttpRequest的execute方法, 其实就是创建一个
            InterceptingRequestExecution, 并调用InterceptingRequestExecution的execute方法
        <3> 调用所有的拦截器的intercept方法, 最后走到InterceptingRequestExecution的execute方
            法的else逻辑
        <4> 利用SimpleClientHttpRequestFactory的createRequest方法创建一个
            SimpleStreamingClientHttpRequest对象, 发送请求

SimpleStreamingClientHttpRequest是HttpRequest的实现类, 所以HttpRequestWrapper的作用就很清晰
了, 对HttpRequest进行了包装:
    public class HttpRequestWrapper implements HttpRequest {
	    private final HttpRequest request;
    }
典型的装饰者模式, 对传入并保存起来的的HttpRequest对象进行了增强, 我们来看看Ribbon中的实现:
public class ServiceRequestWrapper extends HttpRequestWrapper {
	private final ServiceInstance instance;
	private final LoadBalancerClient loadBalancer;

	@Override
	public URI getURI() {
		URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
		return uri;
	}
}

重写了getURI方法, 利用LoadBalancerClient的reconstructURI方法来重构URI, 其实这个方法很简单, 我
们在请求一个服务的时候, host这一块放置的服务Id, 而不是真正的域名, reconstructURI方法就是用来将
经过负载均衡器挑选出来的服务域名替换掉这个host, 至于LoadBalancerClient是干啥的我们之后再分析
```

二、LoadBalancerRequest分析
```java
LoadBalancerRequest是一个函数式接口, 我们先来看看其接口:
    public interface LoadBalancerRequest<T> {
        T apply(ServiceInstance instance) throws Exception;
    }
传入一个ServiceInstance, 返回一个泛型T, 在SpringCloud整合的Ribbon中, 泛型T就是RestTemplate
中的响应实体类ClientHttpResponse, 所以apply方法其实就说执行一个负载均衡请求的, 我们来看看其实现
public LoadBalancerRequest<ClientHttpResponse> createRequest(
        final HttpRequest request, final byte[] body,
        final ClientHttpRequestExecution execution) {
    return instance -> {
        // 简略了一部分代码
        HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
                this.loadBalancer);
        return execution.execute(serviceRequest, body);
    };
}

通过createRequest来创建一个内部类, 其类型是LoadBalancerRequest, 我们来看看这个内部类中实现的
apply方法, 最后通过ClientHttpRequestExecution的execute方法来执行请求, 回想第一篇文章对
RestTemplate中拦截器基于责任链模式下的实现, 当通过迭代器发现所有拦截器均被执行完毕的时候, 就利用
execute方法中传入的HttpRequest来真正的发送请求, 可以看到, 其实就说利用ServiceRequestWrapper来
发送请求的, 但是其里面重写了getURI方法, 以及持有一个原生的HttpRequest对象的引用, 所以真正的原理
就说利用重写的getURI来获取真正发送请求的uri, 最后利用原生的HttpRequest对象进行请求的发送

createRequest方法是LoadBalancerRequestFactory中的, 我们再来看看LoadBalancerRequestFactory的
源码:
// 简略了一部分代码
public class LoadBalancerRequestFactory {
	private LoadBalancerClient loadBalancer;
	public LoadBalancerRequest<ClientHttpResponse> createRequest(
			final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) {
		return instance -> {
			HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
					this.loadBalancer);
			return execution.execute(serviceRequest, body);
		};
	}
}
```

三、LoadBalancerClient分析
```java
LoadBalancerClient是真正用来执行负载均衡请求的组件, 在其中整合了负载均衡器LoadBalancer, 并提供
了一些工具方法来实现负载均衡请求的真正调用, 我们先来看看这个接口:
public interface LoadBalancerClient extends ServiceInstanceChooser {
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

	<T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException;

	URI reconstructURI(ServiceInstance instance, URI original);
}

execute方法用来真正的执行请求,里面的主要逻辑是, 根据serviceId获取负载均衡器, 没错, 每类服务都有
其对应的负载均衡器, 回想起我们在第二篇文章中讲解的SpringClientFactory, 每个服务名称都会在Map中
有自己的Spring上下文环境, 在这个上下文环境中保存了该服务的相关配置, 在之后的分析中我们可以看到
获取负载均衡器LoadBalancer就是根据服务名称去这个Map中找到对应的Spring上下文环境, 进而找到对应的
LoadBalancer的

reconstructURI用来构建真正请求的url的, 我们在请求一个服务的时候, host这一块放置的服务Id, 而不
是真正的域名, reconstructURI方法就说用来将经过负载均衡器挑选出来的服务域名替换掉这个host

LoadBalancerClient接口只有一个实现类RibbonLoadBalancerClient, 我们来看看其execute方法调用链的
实现:
public class RibbonLoadBalancerClient implements LoadBalancerClient {
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		Server server = getServer(loadBalancer, hint);
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
		return execute(serviceId, ribbonServer, request);
	}
    
    public <T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException {
		Server server = null;
		if (serviceInstance instanceof RibbonServer) {
			server = ((RibbonServer) serviceInstance).getServer();
		}
        T returnVal = request.apply(serviceInstance);
        return returnVal;
	}
}

分析: 
    第一个execute方法主要做了三件事情:
        <1> 利用服务名serviceId去SpringClientFactory找到对应的该服务对应的Spring上下文环境,
            再从Spring上下文环境中找到对应的负载均衡器(ZoneAwareLoadBalancer)
        <2> 利用找到的负载均衡器的IRule负载均衡策略, 从保存的服务列表中找到一个合适的Server,
            即getServer方法的实现
        <3> 将这个Server封装成RibbonServer, RibbonServer在原来的Server基础上保存了serviceId,
            即服务名称, 比如购物车服务
        相信有了上一篇文章的分析, 对这三步的理解应该会非常的轻松

    当RibbonServer获取到了以后, 就调用第二个execute方法, 第二个execute方法其实也很简单, 就是
    获取到原生的Server, 然后调用LoadBalancerRequest的apply方法, 而apply方法其实最终会调用到
    我们第二小节LoadBalancerRequestFactory中createRequest方法中创建的LoadBalancerRequest的实
    现类(内部类)中, 最后利用ClientHttpRequestExecution的execute方法来执行调用链
```

四、LoadBalancerInterceptor分析
```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		return this.loadBalancer.execute(serviceName,
				this.requestFactory.createRequest(request, body, execution));
	}
}

分析: 
    LoadBalancerInterceptor是Ribbon和RestTemplate整合的入口, 通过拦截器来实现负载均衡请求的发
    送, 可以看到, 拦截器方法intercept就是利用LoadBalancerClient来执行请求的发送的, 通过
    LoadBalancerRequestFactory的createRequest方法来对拦截器中传入的HttpRequest进行包装, 包装
    的实现类是ServiceRequestWrapper, 该类重写了getURI方法, 利用LoadBalancerClient的
    reconstructURI方法来生成最终请求的url
```

五、总结
```
经过四篇文章由一个个组件开始对Ribbon的源码以及RestTemplate源码进行了分析, 相信大家对Ribbon及
RestTemplate的原理有了更加深刻的认识, 下面对RestTemplate发送一个请求的执行流程做一个简单的总结
    <1> RestTemplate通过doExecute作为入口, 利用createRequest方法创建了一个
        InterceptingClientHttpRequest请求对象, 该对象的executeInternal方法利用
        InterceptingRequestExecution来完成请求的执行
    <2> InterceptingRequestExecution的execute方法基于责任链模式, 对所有的拦截器进行执行后, 最
        终调用传入的HttpRequest进行请求的发送, 在Ribbon中, 最终传入的是ServiceRequestWrapper
    <3> 当所有的拦截器都执行完毕后, 最终走到了nterceptingRequestExecution的execute方法的else
        里面, 里面两行核心的代码如下:
            ClientHttpRequest delegate = 
                requestFactory.createRequest(request.getURI(), method);
            return delegate.execute();
        而ServiceRequestWrapper重写getURI方法利用负载均衡器来获取真正的url
    <4> LoadBalancerInterceptor是Ribbon与RestTemplate整合的拦截器, 利用LoadBalanceClient来
        进行负载均衡url的获取, 在该方法中, 主要有三步:
        <1> 利用服务名serviceId去SpringClientFactory找到对应的该服务对应的Spring上下文环境,
            再从Spring上下文环境中找到对应的负载均衡器(ZoneAwareLoadBalancer)
        <2> 利用找到的负载均衡器的IRule负载均衡策略, 从保存的服务列表中找到一个合适的Server,
            即getServer方法的实现
        <3> 将这个Server封装成RibbonServer, RibbonServer在原来的Server基础上保存了serviceId,
            即服务名称, 比如购物车服务 
```
