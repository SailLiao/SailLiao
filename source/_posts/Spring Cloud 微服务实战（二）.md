---
title: Spring Cloud 微服务实战（二）
date: 2021-02-04 14:49:56
tags: Spring Cloud
cover: https://img9.51tietu.net/pic/2019-091403/tdia2ondsjotdia2ondsjo.jpg
---

书接上回

## Spring Cloud Ribbon

负载均衡有服务端的，例如硬件的F5, 软件的 Nginx， Ribbon 是客户端负载均衡，由客户端来选取调用谁

在深入 Ribbon 前，来学习下已经用过的东西 RestTemplate

### RestTemplate GET 请求

get 主要有两种 

* getForEntity 返回的是 ResponseEntity

例如

```java

RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://USERSERVICE/user?name={1} ", String. class, "didi") ;
String body = responseEntity.getBody();

// 如果想返回对象

RestTemplate restTemplate = new RestTemplate();
ResponseEntity<User> responseEnti七y = restTemplate.getForEntity("http://USERSERVICE/user?name={l}", User.class, "didi");
User body = responseEntity.getBody();

```

另外还有3种不同的重载
1. getForEntity(String url, Class<T> responseType, Object... uriVariables)
2. getForEntity(String url, Class<T> responseType, Map<String, ?> uriVariables)
3. getForEntity(URI url, Class<T> responseType)

这些在类 RestTemplate 中都能找到

* getForObject 函数 

主要用来包装对象

也有三种不同的重载
1. getForObject(String url, Class<T> responseType, Object... uriVariables)
2. getForObject(String url, Class<T> responseType, Map<String, ?> uriVariables)
3. getForObject(URI url, Class<T> responseType)

感觉是不是和上面一样的

### RestTemplate POST 请求

与 GET 大差不差，就不再多讲了，有 postForEntity、postForObject、postForLocation (以POST请求提交资源， 并返回新资源的URI)

除了 GET POST 还有 PUT DELETE 等

## @LoadBalanced 源码解析

在 @LoadBalanced 注释里可以看见是让 RestTemplate 被标记为 LoadBalancerClient 来使用 ?? 

接口 LoadBalancerClient 里面有这些方法

```java
// 根据服务ID来选择服务
ServiceInstance choose(String serviceId);

// 根据服务ID，来执行请求
<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;

// 将服务替换为 IP 和 端口 的形式
URI reconstructURI(ServiceInstance instance, URI original);

```

在上篇文章中我们知道，负载均衡的实现是通过拦截器来完成的，在 RibbonLoadBalancerClient 中有 execute 方法

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
            isSecure(server, serviceId),
            serverIntrospector(serviceId).getMetadata(server));

    return execute(serviceId, ribbonServer, request);
}

// getServer

protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // Use 'default' on a null hint, or just pass it on?
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}

```

这里根据 serviceId 返回的是 com.netflix.loadbalancer.ILoadBalancer， getServer 中 并没有使用 LoadBalancerClient 接口中的 choose 函数，而是 Netflix Ribbon 自身的 ILoadBalancer 接口中定义的 chooseServer 函数，我们来看看 ILoadBalancer 类

```java
// 增加服务实例
public void addServers(List<Server> newServers);
// 选择服务实例
public Server chooseServer(Object key);
// 将服务实例标记为下线，这样负载均衡器就不会选择这个服务了
public void markServerDown(Server server);
// 获取当前正常的服务实例
public List<Server> getReachableServers();
// 获取全部的服务实例
public List<Server> getAllServers();
```
都是见名知意的接口，Server 对象定义是一个传统的服务端节点，在该类中存储了服务端节点的一些元数据信息， 包括 host、 port 以及一些部署信息等。下面是 ILoadBalancer 一些 diagrams

![](1.png)

那么 ribbon 在 整合 Spring cloud 的时候默认使用的是那个均衡器呢?在类 RibbonClientConfiguration 中能找到

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
        IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
    if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
        return this.propertiesFactory.get(ILoadBalancer.class, config, name);
    }
    return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList, serverListFilter, serverListUpdater);
}
```
默认是 ZoneAwareLoadBalancer 

## 负载均衡器

通过之前的分析， 我们已经对Spring Cloud如何使用 Ribbon有了基本的了解。 虽然 Spring Cloud 中定义了 LoadBalancerClient 工作为负载均衡器的通用接口， 并且针对 Ribbon 实现了RibbonLoadBalancerClient， 但是 它在具体实现客户端负载均衡时，是通过 Ribbon 的 ILoadBalancer 接口实现的。 在上一节进行分析时候， 我们对该接口的实现结构已经做了一些简单的介绍， 下面我们根据 ILoadBalancer 接口的实现类逐个看看它是如何实现客户端负载均衡的。

### AbstractLoadBalancer

AbstractLoadBalancer 是  ILoadBalancer 实现的抽象类，里面定义了一个服务的分组枚举，实现了 chooseServer, 新增了两个抽象函数
```java
public enum ServerGroup{
    // 所有服务
    ALL,
    // 正常服务
    STATUS_UP,
    // 不正常服务
    STATUS_NOT_UP        
}

public Server chooseServer() {
    return chooseServer(null);
}

// 根据 serverGroup 列出此 LoadBalancer 知道的服务
public abstract List<Server> getServerList(ServerGroup serverGroup);

// 获取与 LoadBalancer 相关的统计信息
public abstract LoadBalancerStats getLoadBalancerStats();   

```

### BaseLoadBalancer

Ribbon 负载均衡器的基础实现类，在该类中定义了很多关于负载均衡器相关的基础内容。比较关键的是

```java

// 全部server
@Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> allServerList = Collections.synchronizedList(new ArrayList<Server>());

// 正常的server
@Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> upServerList = Collections.synchronizedList(new ArrayList<Server>());

// 存储负载均衡器各服务实例属性和统计信息
protected LoadBalancerStats lbStats;

// 检查服务实例是否正常服务
protected IPing ping = null;

// 负载均衡规则，默认线性负载均衡
private final static IRule DEFAULT_RULE = new RoundRobinRule();
protected IRule rule = DEFAULT_RULE;

...

```

### DynamicServerListLoadBalancer

实现了服务实例清单在运行期的动态更新能力；同时， 它还具备了对服务实例清单的过滤功能， 也就是说， 我们可以通过过滤器来选择性地获取一批服务实例清单。在成员变量中可以看见 
```java
volatile ServerList<T> serverListImpl;
```

泛型 T 现定于 Server 的子类，ServerList 的接口定义如下

```java
// 获取初始化的服务清单
public List<T> getInitialListOfServers();
// 获取更新的服务清单
public List<T> getUpdatedListOfServers();   
```

![](2.png)

可以看出 ServerList 有多个实现，那么在 DynamicServerListLoadBalancer 中使用的是那个呢，在类 EurekaRibbonClientConfiguration 中，能找到

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config, Provider<EurekaClient> eurekaClientProvider) {
    if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
        return this.propertiesFactory.get(ServerList.class, config, serviceId);
    }
    DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(config, eurekaClientProvider);
    DomainExtractingServerList serverList = new DomainExtractingServerList(discoveryServerList, config, this.approximateZoneFromHostname);
    return serverList;
}
```
创建的是 DomainExtractingServerList 

### ZoneAwareLoadBalancer

ZoneAwareLoadBalancer 默认会用轮询的方式来访问

## 负载均衡策略

通过 IRule 能看出有很多实现

![](3.png)

### RandomRule

该策略实现了从服务实例清单中随机选择 一个服务实例的功能。

### RoundRobinRule

线性轮询的方式。

### RetryRule

该策略实现了一个具备重试机制的实例选择功能。

### WeightedResponseTimeRule

该策略是对 RoundRobinRule 的扩展， 增加了根据实例的运行情况来计算权重， 并根据权重来挑选实例， 以达到更优的分配效果。



