一、RibbonClient的使用
```java
Ribbon作为一个负载均衡组件, 里面有一个个的负载均衡策略, 而这些策略的公共接口是IRule, 我们在开始
学习Ribbon的时候, 必然会接触到这个IRule接口, 通过往Spring容器中注入一个IRule接口的实现类, 可以
改变Ribbon默认的负载均衡策略(默认是轮询), 假设有这么一种场景, 我们有一个购物车服务CartService以
及一个订单服务OrderService, 我期望调用端在调用CartService的时候, 负载均衡策略为随机策略, 而调用
订单服务还是采用默认的策略, 那么我们可能会这样做:

@RibbonClient( value = "CartService", configuration = MyRibbonConfiguration.class )
@Configuration
public class SpringConfig () {}

public class MyRibbonConfiguration {
    @Bean
    public IRule iRule () {
        return new RandomRule();
    }
}

通过上面的方式, 我们就使得对于CartService的调用, 负载均衡策略就变成了随机策略了, 接下来我们就来
分析下Ribbon到底是怎么来区分不同服务的负载均衡相关配置的
```

二、RibbonClientConfigurationRegistrar源码分析

2.1、@Import注解和RibbonClientConfigurationRegistrar
```java
@Import(RibbonClientConfigurationRegistrar.class)
public @interface RibbonClient {
	String value() default "";

	String name() default "";

	Class<?>[] configuration() default {};
}

分析: 如果对Spring源码有所了解的话, 就可以知道, @Import注解有三种使用方式, 第一种是value为一个
Spring的配置类, 比如一个类A里面有一个@Bean方法, 当@Import注解中的value为这个类A的时候, 就会使得
@Bean返回的对象被注入到Spring容器中

第二种是value为ImportSelector的实现类, 这个函数式接口有一个selectImports方法, 返回的是一个
String类型的数组, 这个数组中应该是一个个的全路径类名, Spring会通过反射的方式将这些类注入到容器
中

第三种是value为ImportBeanDefinitionRegistrar接口的实现类, 这个函数式接口有一个
registerBeanDefinitions方法, 该方法会有一个BeanDefinitionRegistry参数, 通过这个参数可以使得我
们手动的注入Beanefinition到容器中

对于Import注解的解析是在一个BeanFactoryPostProcessor中完成的, 这些都是跟Spring相关的了, 如果对
这些都熟悉的话, 那么我们接下来的分析大家就会觉得很简单了

RibbonClientConfigurationRegistrar属于第三种, 这个类完成的作用就是, 解析RibbonClient注解或者
RibbonClients注解, 将一个个的RibbonClient注解中的内容封装成一个RibbonClientSpecification对象,
并注入到容器中

public class RibbonClientSpecification implements NamedContextFactory.Specification {
	private String name;
	private Class<?>[] configuration;
}

可以看到, 其实就是将RibbonClient中的name和configuration保存成RibbonClientSpecification的属性
而已, 接下来我们分析RibbonClientConfigurationRegistrar中的registerBeanDefinitions方法
```

2.2、registerBeanDefinitions第一段代码
```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    Map<String, Object> attrs = metadata
            .getAnnotationAttributes(RibbonClients.class.getName(), true);
    if (attrs != null && attrs.containsKey("value")) {
        AnnotationAttributes[] clients = (AnnotationAttributes[]) attrs.get("value");
        for (AnnotationAttributes client : clients) {
            registerClientConfiguration(registry, getClientName(client),
                    client.get("configuration"));
        }
    }
}

分析：
    首先从@Import(RibbonClientConfigurationRegistrar.class)标注的类或者该注解的父注解标注的类
    中查找是否有RibbonClients注解, 如果有, 并且value不为空, 说明RibbonClients注解中有
    RibbonClient注解, 于是将这些注解一个个的取出来, 利用getClientName方法对RibbonClient注解
    进行解析, 得到里面的name或者value, 以及RibbonClient注解中的configuration, 进而通过
    registerClientConfiguration构造一个RibbonClientSpecification对象并注入到Spring容器中

private String getClientName(Map<String, Object> client) {
    if (client == null) {
        return null;
    }
    String value = (String) client.get("value");
    if (!StringUtils.hasText(value)) {
        value = (String) client.get("name");
    }
    if (StringUtils.hasText(value)) {
        return value;
    }
}

分析: 
    很简单, 就是取得RibbonClient注解中的name或者value

private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(RibbonClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(name + ".RibbonClientSpecification",
            builder.getBeanDefinition());
}

分析:
    生成一个RibbonClientSpecification的BeanDefinition, 设置构造参数name为RibbonClient中的
    name或者value, configuration为RibbonClient中的configuration属性, 这样以后一旦Spring根据
    这个BeanDefinition创建了bean对象后, 里面属性中的name和configuration就是RibbonClient中的
    name/value和configuration了

    注入到Spring容器中的时候, RibbonClient中的name/value 加上.RibbonClientSpecification作为
    beanName, 指定了构造参数值后的RibbonClientSpecification作为bean

在我们最开始的那个例子中, 其实就是注入了一个beanName为CartService.RibbonClientSpecification,
bean实例如下样子的bean到Spring容器:
public class RibbonClientSpecification implements NamedContextFactory.Specification {
	private String name = "CartService";
	private Class<?>[] configuration = MyRibbonConfiguration.class
}
```

2.3、registerBeanDefinitions第二段代码
```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    if (attrs != null && attrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        }
        else {
            name = "default." + metadata.getClassName();
        }
        registerClientConfiguration(registry, name,
                attrs.get("defaultConfiguration"));
    }
}

分析: 如果RibbonClients注解中配置了defaultConfiguration, 即默认的配置, 我们结合下面这个例子来
说明这段代码的功能:
@RibbonClients(defaultConfiguration = EurekaRibbonClientConfiguration.class)
public class RibbonEurekaAutoConfiguration {}

当RibbonClients注解被解析的时候, 里面的@Import方法触发第二段代码的时候, 就会注入一个beanName为
default.RibbonEurekaAutoConfiguration.RibbonClientSpecification, bean为如下样子的bean到
Spring容器中:
public class RibbonClientSpecification implements NamedContextFactory.Specification {
	private String name = "default.RibbonEurekaAutoConfiguration";
	private Class<?>[] configuration = RibbonEurekaAutoConfiguration.class
}
```

2.4、registerBeanDefinitions第三段代码
```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
    Map<String, Object> client = metadata
            .getAnnotationAttributes(RibbonClient.class.getName(), true);
    String name = getClientName(client);
    if (name != null) {
        registerClientConfiguration(registry, name, client.get("configuration"));
    }
}

分析:
    第三段代码就很简单了, 就是解析类上的RibbonClient注解
```

2.5、小小的总结
```
RibbonClientConfigurationRegistrar的registerBeanDefinitions方法, 一共完成了三个事情, 第一个
是解析@RibbonClients注解中的RibbonClient注解, 第二个是解析RibbonClients注解中的
defaultConfiguration属性, 第三个是解析RibbonClient注解, 第一个和第三个是不一样的, 前者是在
@RibbonClients中的value中, 而后者是直接标注在类上的

解析完毕后会封装成一个个的RibbonClientSpecification对象注入到Spring容器中, 该对象的name就是
@RibbonClient注解中的name / value, configuration就是@RibbonClient注解中的configuration

那么这一个个的RibbonClientSpecification对象到底有什么作用呢??接下来就要分析
SpringClientFactory这个类的作用了
```

三、SpringClientFactory源码分析

3.1、SpringClientFactory的创建
```java
public class RibbonAutoConfiguration {
	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>(); 

    @Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}    
}

在RibbonAutoConfiguration这个类中, 往Spring容器中注入了一个SpringClientFactory, 并且从Spring
容器中找到所有的RibbonClientSpecification, 并放入到SpringClientFactory中

SpringClientFactory, 是一个工厂类, 根据javadoc中的描述, 它的作用是: 
    创建客户端, 负载均衡器和客户端配置实例的工厂, 它为每个客户端名称创建一个Spring
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {}

先来看看其父类NamedContextFactory的作用吧, 根据javadoc的描述:
    创建一组次级的Spring容器上下文，可以使得每个次级Spring容器上下文定义一组专门的配置bean

用通俗的话来讲, 就是NamedContextFactory中会有一个这样的Map:
    Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();
使得每类服务都可以拥有自己独立的Spring上下文环境来保存该类服务独有的配置, 比如CartSevice可以作为
这个map的key, 对应一个AnnotationConfigApplicationContext, 在这个context中保存CartService独有
的配置
```

3.2、NamedContextFactory源码分析
```java
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
    private Map<String, AnnotationConfigApplicationContext> contexts 
                                        = new ConcurrentHashMap<>();
	private Map<String, C> configurations = new ConcurrentHashMap<>();
	private ApplicationContext parent;
	private Class<?> defaultConfigType;
}

首先分析四个属性, 第一个contexts表示不同的key可以拥有自己独立的Spring上下文, configurations表示
不同的服务可以有自己独有的NamedContextFactory.Specification, 在上面我们分析RibbonClients原理的
时候, 可以看到最后会创建一个个的RibbonClientSpecification, 而RibbonClientSpecification的父类
就是NamedContextFactory.Specification, 所以可以理解为, 不同类型的服务对应的负载均衡配置就存储
在这个configurations中, 在最上面的例子的条件下, 这个configurations中会保存一条记录为:
    key: CartService
    value: RibbonClientSpecification{ 
                 name: "CartService", configuration: MyRibbonConfiguration.class
           }

parent表示父容器, 即项目初始化所创建的Spring容器, 我们真正使用容器

defaultConfigType为RibbonClientConfiguration, 在我们创建SpringClientFactory的时候会设置进去,
表示Ribbon客户端默认的配置

public class RibbonAutoConfiguration {
	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>(); 

    @Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}    
}

public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
    public void setConfigurations(List<C> configurations) {
        for (C client : configurations) {
            this.configurations.put(client.getName(), client);
        }
    }

}

通过上面的代码, 我们可以验证, 最终NamedContextFactory的configurations对应的Map存储的就是通过
扫描RibbonClient等注解创建的RibbonClientSpecification对象

public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
    public <T> T getInstance(String name, Class<T> type) {
        AnnotationConfigApplicationContext context = getContext(name);
        if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
                type).length > 0) {
            return context.getBean(type);
        }
        return null;
    }

    protected AnnotationConfigApplicationContext getContext(String name) {
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}
}

分析: 通过name从Map<String, AnnotationConfigApplicationContext> contexts 这个Map中找到对应
的Spring上下文, 然后从这个Spring上下文中获取type对应的bean对象, 这就是getInstance方法的作用,
如果这个Spring上下文不存在则会创建并放入到Map中, createContext就不分析了, 有兴趣的话可以看下,
其实就是判断configurations中存不存在name对应的配置, 如果存在就注入到这个次级的上下文中, 其次,
注入configurations中以default开头的配置到这个次级的上下文中(default开头的配置其实就是
RibbonClients中定义的defaultConfiguration属性值), 最后调用context.refresh方法刷新Spring容器
```

3.3、SpringClientFactory源码分析
```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
    public SpringClientFactory() {
		super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
	}

    public <C extends IClient<?, ?>> C getClient(String name, Class<C> clientClass) {
		return getInstance(name, clientClass);
	}

    public ILoadBalancer getLoadBalancer(String name) {
		return getInstance(name, ILoadBalancer.class);
	}

    public RibbonLoadBalancerContext getLoadBalancerContext(String serviceId) {
		return getInstance(serviceId, RibbonLoadBalancerContext.class);
	}
}

分析: 
    可以看到, 其实SpringClientFactory就是对父类的getInstance进行了一下封装而已, 最终其实就是
    根据name从父类的所有次级Spring上下文中找到对应的上下文(如果不存在则创建), 然后从上下文中找到
    对应的bean对象, 其次在构造方法中指定了父类的defaultConfigType为RibbonClientConfiguration
```

四、总结
```java
RibbonClients注解和RibbonClient注解通过RibbonClientConfigurationRegistrar来进行解析, 最终会
以RibbonClientSpecification类对象的形式存储在Spring容器中, 在RibbonAutoConfiguration中, 创建
了SpringClientFactory, 该对象提供了一种命名空间形式的配置保存方式, 不同名称拥有不同的Spring上下
文环境, 在该上下文环境中保存了对应名称的配置
```
