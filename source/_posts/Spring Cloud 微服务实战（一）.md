---
title: Spring Cloud 微服务实战（一）'
date: 2021-02-02 14:51:22
tags: Spring Cloud
cover: https://img9.51tietu.net/pic/2019-091403/0skqbqkfkdi0skqbqkfkdi.jpg
---

看书做的简单笔记，书中有些方式已经过时，用的新的版本来做的，主要是学习的目的

## Spring boot

在开始之前简单介绍下 Spring Boot

在项目的 pom.xml 文件中 包含了下面两项。
* spring-boot-starter-web : 全栈Web开发模块， 包含嵌入式Tomcat、 SpringMVC。
* spring-boot-starter士est: 通用测试模块， 包含JUnit、 Hamcrest、 Mockito 。

这里所引用的web和test 模块，在SpringBoot 生态中被称为 **Starter POMs**。
Starter POMs 是一系列轻便的依赖 包， 是一套一站式的Spring相关技术的解决方案。 
开发者在使用和整合模块时， 不必再去搜寻样例代码中的依赖配置来复制使用， 只需要引入对应的模块包即可 。 
比如：

* 开发Web应用的时候， 就引入 spring-boot-starter-web
* 希望应用具备访问数据库能力的时候， 那就再引入 spring-boot-starter-jdbc 
* 或是更好用的 spring-boot-starter-data-jpa

最后， 项目构建的 build部分， 引入了Spring Boot的Maven插件
```xml
<build>
    <plugins>
        <plugin>
            <groupid>org.springframework.boot</groupid>
            <artifactid>spring-boot-maven-plugin</artifactid>
        </plugin>
    </plugins>
</build> 
```
该插件非常实用，可以帮助我们方便地启停应用，这样在开发时就不用每次去找主类或是打包成jar来运行微
服务， 只需要通过 **mvn spring-boot:run** 命令就可以快速启动Spring Boot应用。

### 实现RESTfulAPI
很简单不再赘述
```java

@RestController
public class HelloController {

    @RequestMapping ("/hello")
    public String index() {
        return "Hello World";
    }
}
```

可以配置下端口之类的，默认是 8080， 默认的访问路径是 '', 在控制台也会有体现

```log
Tomcat started on port(s): 8080 (http) with context path ''
```

通过浏览器访问http://localhost:8080/hello, 我们可以看到 返回了预期结果： Hello World。

### 进行单元测试
```java

import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status; 


@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = HelloApplication.class)
@WebAppConfiguration
public class HelloApplicationTests (

    private MockMvc mvc;

    @Before
    public void setOp() throws Exception {
        mvc = MockMvcBuilders.standaloneSetup(new HelloController()).build() ;
    }

    @Test
    public void hello() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
        .andExpect(status().isOk())
        .andExpect(content().string(equalTo("Hello World")));
    }
}
```
代码解析如下。
* @RunWith(SpringJUnit4ClassRunner.class): 引入Spring对JUnit4的支持。
* @SpringApplicationConfiguration(classes=HelloApplication.class): 指定Spring Boot的启动类。
* @WebAppConfiguration: 开启Web应用的配置， 用千模拟ServletContext。
* MockMvc对象： 用于模拟调用 Controller的接口发起请求， 在@Test定义的hello测试用例中， perform函数执行 一次请求调用， accept用于执行接收的数据类型，andExpect用于判断接口返回的期望值。
* @Before: JUnit中定义在测试用例@Test内容执行前预加载的内容， 这里用来初始化对HelloController的模拟 。


### 配置加载方式
目前有 properties 和 yml 方式，yml 方式不能通过 @PropertySource 注解来加载配置，但是 yml 的方式加载是有序的。总体说来 yml 好
在配置中使用随机数
```
${random}的配置方式主要有以下几种， 读者可作为参考使用。
＃随机字符串
com.didispace.blog.value=${random.value}
＃随机int
com.didispace.blog.number=${random.int}
＃随机long
com.didispace.blog.bignumber=${random.long}
# 10以内的随机数
com.didispace.blog.test1=${random.int(l0)}
# 10-20的随机数
com.didispace.blog.test2=${random.int[l0,20]}
```

### SpringBoot对数据文件的加载机制
配置的优先级

1. 在命令行中传入的参数。
2. SPRING APPLICATION JSON中的属性。 SPRING_APPLICATION—JSON是以JSON格式配置在系统环境变量中的内容。
3. java:comp/env中的JNDI 属性。
4. Java的系统属性， 可以通过System.getProperties()获得的内容。
5. 操作系统的环境变量 。
6. 通过random.*配置的随机属性。
7. 位于当前应用 jar 包之外，针对不同{profile}环境的配置文件内容，例如application-{profile}.properties或是YAML定义的配置文件。
8. 位于当前应用 jar 包之内，针对不同{profile}环境的配置文件内容，例如application-{profile}.properties或是YAML定义的配置文件。
9. 位于当前应用jar包之外的application.proper巨es和YAML配置内容。
10. 位于当前应用jar包之内的app口ca巨on.proper巨es和YAML配置内容。
11. 在@Configura巨on注解修改的类中，通过@PropertySource注解定义的属性。
12. 应用默认属性，使用SpringApplication.se七DefaultProper巨es 定义的内容。

优先级按上面的顺序由高到低，数字越小优先级越高

中第7项和第9项 都是从应用jar包之外读取配置文件，所以，实现外部
化配置的原理就是从此切入，**为其指定外部配置文件的加载位置来取代jar包之内的配置内容**。 
通过这样的实现，我们的工程在配置中就变得非常干净，只需在本地放置开发需要的
配置即可，而不用关心其他环境的配置，由其对应环境的负责人去维护即可。

### actuator

spring-boo七-starter-actuator 提供一系列用千监控的端点，
```xml
<dependency>
    <groupid>org.springframework.boot</groupid>
    <artifactid>spring-boot-starter-actuator</artifactid>
</dependency>
```

* /autoconfig
该端点用来获取应用的自动化配置报告， 其中包括所有自动化配置的候选项。

* /beans
该端点用来获取应用上下文中创建的所有Bean。

* /configprops
该端点用来获取应用中配置的属性信息报告。

* /env
该端点与/configprops不同它用来获取应用所有可用的环境属性报告。

* /mappings
该端点用来返回所有Spring MVC的控制器映射关系报告。

* /info
该端点用来返回一些应用自定义的信息。 默认清况下， 该瑞点只会返回 一个空的JSON内容。
我们可以在application.properties配置文件中通过info前缀来设置一些属性， 比如下面这样：
```properties
info.app.name=spring-boot-hello 
info.app.version=vl.0.0 
```
再访问/info端点我们可以得到包含了上面我们在应用中自定义的两个参数。

* /metrics
该端点用来返回当前应用的各类重要度量指标，比如内存信息、线程信息、垃圾回收信息等。

* /health
该端点在一开始的示例中 我们已经使用过了，它用来获取应用的各类 健康指标信息。

* /dump
该端点用来暴露程序运行中的线程信息。它使用 java.lang.managernentThreadMXBean 的 dumpAllThreads 方法来返回所有含有同步信息的活动线程详情。

* /trace
该端点用来返回基本的 HTTP 跟踪信息。 默认情况下， 跟踪信息的存储采用org.springfrarnework.boot.actuate.trace.InMernoryTraceRepository实现的内存方式， 始终保留最近的100条请求记录。


## Eureka

主要负责完成微服务架构中的服务治理功能

```xml

<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.5.RELEASE</version>
</parent>

<properties>
    <java.version>1.8</java.version>
    <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
</properties>

<!-- spring-cloud -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- eureka-server -->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```


需要注意的是 Spring boot 的版本和 Spring cloud 的版本, 也简单学习下 dependencyManagement 这个标签

> Maven约定优于配置的理解，dependencies 中的jar直接加到项目中，管理的是依赖关系（如果有父pom,子pom,则子pom中只能被动接受父类的版本）；dependencyManagement 主要管理版本，对于子类继承同一个父类是很有用的，集中管理依赖版本不添加依赖关系，对于其中定义的版本，子pom不一定要继承父pom所定义的版本。


区别：
1. dependencies 即使在子项目中不写该依赖项，那么子项目仍然会从父项目中继承该依赖项（全部继承）
2. dependencyManagement 里只是声明依赖，并不实现引入，因此子项目需要显示的声明需要用的依赖。如果不在子项目中声明依赖，是不会从父项目中继承下来的；只有在子项目中写了该依赖项，并且没有指定具体版本，才会从父项目中继承该项，并且version和scope都读取自父pom;另外如果子项目中指定了版本号，那么会使用子项目中指定的jar版本。



通过@EnableEurekaServer 注解启动一个服务注册中心提供给其他应用进行对话。
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }

}
```
在默认设置下， 该服务注册中心也会将自己作为客户端来尝试注册它自己
```yml
eureka:
  instance:
    hostname: cloud-monitor-eureka
    prefer-ip-address: true
  client:
    # 不注册自己
    register-with-eureka: false
    # 由于注册中心的职责就是维护服务实例，它并不需要去检索服务， 所以也设置为 false。
    fetch-registry: false 
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

然后访问端口就能看见我们的 Eureka Server 页面了，但是这个时候没有任何客户端注册进来

![](2.png)

现在我们 将一个的 springboot 服务加入到 eureka 服务中去作为服务提供者 maven 配置与上面是一样的

```java

// 启动类上面加上 @EnableEurekaClient 注解
@EnableEurekaClient
@SpringBootApplication
public class DemoProviderApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoProviderApplication.class, args);
	}

}


@RestController
@RequestMapping
public class Controller {

    private final Logger logger = Logger.getLogger(this.getClass().getName());

    // 将 org.springframework.cloud.client.discovery.DiscoveryClient; 注入进来
    @Autowired
    private DiscoveryClient client;

    @GetMapping(value = "/text")
    public String text() {
        logger.info("调用了接口");
        return "Hello Word";
    }

}
```

在配置文件中指定要连接的 Eureka 的服务地址

```
#服务注册中心端口号
server.port=1112

spring.application.name=hello-service

eureka.client.serviceUrl.defaultZone=http://localhost:1111/eureka/
```

然后启动服务，会发现，已经有服务了，名称是 HELLO-SERVICE， 来自于 windows10.microdone.cn:hello-service:1112

![](3.png)

同理我们再启动一个 provider，指定不同的端口，能看到多个provider是用逗号隔开的

![](4.png)

接下来，我们来做一个服务的消费者，一样的创建一个新的 spring boot 项目，maven 配置与上面的一样

```java

// 启动类需要注入 RestTemplate, 使用 @EnableEurekaClient 注解
@EnableEurekaClient
@SpringBootApplication
public class DemoConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoConsumerApplication.class, args);
    }

    /**
     * @return org.springframework.web.client.RestTemplate
     */
    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

}

// Controller

@RestController
@RequestMapping("index")
public class Controller {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("get")
    public String get() {
        // 我们想请求的远程服务，hello-service 是服务名称，/text 是具体的接口
        String memberUrl = "http://hello-service/text"; 
        String result = restTemplate.getForObject(memberUrl, String.class);
        System.out.println("result: " + result);
        return result;
    }

}

```
启动我们的 consumer， 可以在 Eureka 的 web 界面上看见多了 consumer 

![](5.png)

然后请求消费者提供的接口 http://127.0.0.1:1113/index/get 可以看见我们两个 provider 在交替的打印日志。


### 使用 Ribbon

Ribbon 可以让我们轻松地将面向服务的REST模版请求（RestTemplate）自动转换成客户端负载均衡的服务调用。Ribbon 只是一个工具类框架，但是微服务间的调用，API网关的请求转发等内容，实际上都是通过Ribbon来实现的，包括后续我们将要介绍的Feign，它也是基于Ribbon实现的工具。

其实我们已经使用过 Ribbon 了就是 @LoadBalanced 这个注解，在 **org.springframework.cloud:spring-cloud-netflix-eureka-server** 里面默认也添加了 Ribbon 的依赖,

![](6.png)

Ribbon 实现的方式是给增加了 @LoadBalanced 这个注解的 RestTemplate 添加拦截器，在拦截器里面通过Ribbon选取服务实例，然后将请求的服务器地址中的名称替换成Ribbon选取服务实例的IP和端口

如果没有 @LoadBalanced，RestTemplate 是不具备服务名调用的方式的。那么这个小小的注解，为何如此厉害，我们来深入分析一下它的原理。

关注这个类  **LoadBalancerAutoConfiguration**

在这个类里面 注入了所有加了 @LoadBalanced 注解的 restTemplate 的 Bean 实例

```java

// 注入有 @LoadBalanced 的 RestTemplate
@LoadBalanced
@Autowired(required = false)
private List<RestTemplate> restTemplates = Collections.emptyList();

@Bean
@ConditionalOnMissingBean
public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
    return restTemplate -> {
        List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
        list.add(loadBalancerInterceptor);
        restTemplate.setInterceptors(list); // 加入拦截器
    };
}

// 在拦截器里面 LoadBalancerInterceptor 从

this.requestFactory.createRequest(request, body, execution));

// 点进去 进入到 LoadBalancerRequestFactory 可以看到

HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, this.loadBalancer);

// 点进去 进入到 ServiceRequestWrapper

URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());

// 点进去 进入到 RibbonLoadBalancerClient

return context.reconstructURIWithServer(server, uri);

// 点进去 进入到 LoadBalancerContext 可以看见在根据 server 转换 IP 和 端口

public URI reconstructURIWithServer(Server server, URI original) {
    String host = server.getHost();
    int port = server.getPort();
    ...
    URI newURI = new URI(sb.toString());
    return newURI;         
}

```

后面会专门讲 Ribbon

### 服务注册中心的高可用

可以理解为多个 eureka server , 这个时候服务提供方只需要指定多个 server 就行, 用逗号隔开
```yml
spring:
    application:
        name: hello-service 
eureka:
    client:
        serviceUrl:
            defaultZone: http://peerl:llll/eureka/,http://peer2:lll2/eureka/
```

若此 时断开peer1, 由于compute-service同时也向peer2注册， 因此在peer2上的其他服务依然能访问到hello-service, 从而实现了服务注册中心的高可用。


### 服务注册
“服务提供者” 在启动的时候会通过发送REST请求的方式将自己注册到 EurekaServer 上， 同时带上了自身服务的一些元数据信息。
Eureka Server接收到这个REST请求之后，将元数据信息存储在一个双层结构Map中， 其中第一层的key是服务名， 第二层的key是具体服务的实例名。
在服务注册时，需要确认一下 **eureka.client.register-with-eureka=true**
参数是否正确，该值默认为true。 若设置为false将不会 启动注册操作。

### 服务同步
当服务提供者发送注册请求到一个服务注册中心时，它会将该请求转发给集群中相连的其他注册中心， 从而实现注册中心之间的服务同步

### 服务续约
在注册完服务之后，服务提供者会维护一个心跳用来持续告诉EurekaSe1-ver: "我还活着 ”， 以防止Eureka Server的 “剔除任务 ” 将该服务实例从服务列表中排除出去，我们称该操作为服务续约(Renew)。关千服务续约有两个重要属性，我们可以关注并根据需要来进行调整：
```
eureka.instance.lease-renewal-interval-in-seconds=30 
eureka.instance.lease-expiration-duration-in-seconds=90
```
eureka.instance.lease-renewal-interval-in-seconds   参数用于定义服务续约任务的调用间隔时间，默认为30秒。 
eureka.instance.lease-expiration-duration-in-seconds  参数用于定义服务失效的时间，默认为**90秒**。

### 荻取服务
当我们启动服务消费者的时候， 它会发送一个REST请求给服务注册中心，来获取上面注册的服务清单 。 为了性能考虑， Eureka Server会维护一份只读的服务清单来返回给客户端，同时该缓存清单会每隔30秒更新一次。
```
eureka.client.fetch-registry=true
# 缓存更新时间设置
eureka.client.registry-fetch-interval-seconds=30
```

### 服务调用
在Ribbon中会默认采用轮询的方式进行调用，从而实现客户端的负载均衡。对于访问实例的选择，Eureka中有Region和Zone的概念， 一个Region中可以包含多个Zone, 每个服务客户端需要被注册到 一个Zone中， 所以每个客户端对应一个Region和一个Zone。 在进行服务调用的时候，优先访问同处一个 Zone 中的服务提供方， 若访问不到，就访问其他的Zone

### 服务下线
当服务实例进行正常的关闭操作时， 它会触发一个服务下线的REST请求给Eurka Server, 告诉服务注册中心：“我要下线了”。 服务端在接收到请求之后， 将该服务状态置为下线(DOWN), 并把该下线事件传播出去。

### 失效剔除
服务注册中心并未收到 “服务下线” 的请求。 为了从服务列表中将这些无法提供服务的实例剔除， Eureka Server在启动的时候会创建一个定时任务，默认每隔一段时间（默认为60秒） 将当前清单中超时（默认为90秒）没有续约的服务剔除出去。

### 自我保护
EurekaServer 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%, 如果出现低于的情况（在单机调试的时候很容易满足， 实际在生产环境上通常是由于网络不稳定导致）， EurekaServer会将当前的实例注册信息保护起来， 让这些实例不会过期， 尽可能保护这些注册信息。 但是， 在这段保护期间内实例若出现问题， 那么客户端很容易拿到实际已经不存在的服务实例， 会出现调用失败的清况， 所以客户端必须要有容错机制， 比如可以使用请求重试、 断路器等机制

### 源码分析
从客户端入手，看一下 @EnableDiscoveryClient 的源码
```java
/**
 * Annotation to enable a DiscoveryClient implementation.
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableDiscoveryClientImportSelector.class)
public @interface EnableDiscoveryClient {

	/**
	 * If true, the ServiceRegistry will automatically register the local server.
	 * @return - {@code true} if you want to automatically register.
	 */
	boolean autoRegister() default true;

}
```
从注解可以看出是用来开启 **DiscoveryClient** 的实现的

![](1.png)

DiscoveryClient 是  org.springframework.cloud.client.discovery 包下面的，是spring cloud 提供的接口，定义了发现服务的常用抽象方法，来让不同的三方实现 EurekaDiscoveryClient 是 org.springframework.cloud.netflix.eureka 提供的 Eureka 实现，

EurekaDiscoveryClient 依赖了 Netflix Eureka 的 com.netflix.discovery.EurekaClient 接口, EurekaClient 继承自  com.netflix.discovery.shared.LookupService，我们先看看 com.netflix.discovery.DiscoveryClient

在 com.netflix.discovery.endpoint.EndpointUtils 中可以找到 服务发现
```java
public static Map<String, List<String>> getServiceUrlsMapFromConfig(EurekaClientConfig clientConfig, String instanceZone, boolean preferSameZone) {
        Map<String, List<String>> orderedUrls = new LinkedHashMap<>();
        String region = getRegion(clientConfig);
        String[] availZones = clientConfig.getAvailabilityZones(clientConfig.getRegion());
        if (availZones == null || availZones.length == 0) {
            availZones = new String[1];
            availZones[0] = DEFAULT_ZONE;
        }
        logger.debug("The availability zone for the given region {} are {}", region, availZones);
        int myZoneOffset = getZoneOffset(instanceZone, preferSameZone, availZones);

        String zone = availZones[myZoneOffset];
        List<String> serviceUrls = clientConfig.getEurekaServerServiceUrls(zone);
        if (serviceUrls != null) {
            orderedUrls.put(zone, serviceUrls);
        }
    ...
    return orderedUrls;
}
```

1. 通过 getRegion() 从配置中获得一个 region，如果 region 没设置，就是字符串的 default
2. 然后得到这个 region 下面的 zone, 默认也是 default, **region 与 zone 是一对多** 
3. 多个 zone 用逗号分开
4. 我们使用 Ribbon 实现服务调用的时候，Ribbon 会默认优先访问 **同客户端处于同一个Zone中的服务端实例** , 没有时会访问其他 Zone 中的实例


服务注册是以下的方式

 ```java
 private void initScheduledTasks() {
    ...
    if (clientConfig.shouldRegisterWithEureka()) {
        
        ...

        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        ...

        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
 ```

 在 shouldRegisterWithEureka 内有一个 InstanceInfoReplicator ,这个会执行定时任务

 ```java
 public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
 ```
核心的是 
**discoveryClient.register();**
这一行, 点进去

```java
/**
* Register with the eureka service by making the appropriate REST call.
*/
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```
能看出来是 REST 请求的，传入了 instanceInfo, 该对象就是注册时客户端给服务端的服务的元数据。

服务的获取与续约是在 initScheduledTasks 中通过类 TimedSupervisorTask 来完成。cacheRefresh (CacheRefreshThread) 负责 服务获取，heartbeat (HeartbeatThread) 负责心跳，心跳 和 服务注册在一起的。

续约，心跳 很简单
```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```
能看出来是直接请求了下，服务获取复杂些, 会根据是不是第一次获取发起不同的处理请求 ???

接下来看看 服务中心 是怎么处理这些 REST 接口发送过来的请求 在类 **com.netflix.eureka.resources.ApplicationResource** 中
```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info, @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    ...
    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```
在对一堆参数检验后，会调用 register 方法
```java
@Override
public void register(final InstanceInfo info, final boolean isReplication) {
    handleRegistration(info, resolveInstanceLeaseDuration(info), isReplication);
    super.register(info, isReplication);
}

private void handleRegistration(InstanceInfo info, int leaseDuration, boolean isReplication) {
    log("register " + info.getAppName() + ", vip " + info.getVIPAddress() + ", leaseDuration " + leaseDuration + ", isReplication " + isReplication);
    publishEvent(new EurekaInstanceRegisteredEvent(this, info, leaseDuration, isReplication));
}

```

先会 publishEvent ，将该服务注册的事件传播出去，再调用父类的 register，将 Instanceinfo 插入到一个 ConcurrentHashMap 里面, map 里面第一层是 appName ，第二层 map key 是传入的 InstanceInfo 的id

eureka 对元数据的管理可以查看 Instanceinfo 类，中有个 metadata 的属性，是 ConcurrentHashMap， 我们可以通过eureka.instance.<propertues>=<value>的格式对标准化 元数据直接进行配置
比如
**eureka.instance.metadataMap.zone=shanghai**

eureka 因为是 htpp 的，所以能跨平台支持，比如 eureka-js-client、python-eureka

## 总结
简单的介绍了 Sring boot 和 Eureka ，在 Eureka的服务治理体系中， 主要分为服务端与客户端两个不同的角色， 服务端为服务注册中心， 而客户端为各个提供接口的微服务应用。注册中心的高可用、服务的注册、续约、心跳、服务的获取等。


## 来源

[Spring Cloud 微服务实战](https://book.douban.com/subject/27025912/)