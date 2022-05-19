---
title: Spring 的循环依赖 
date: 2022-04-15 16:12:54
tags: Spring
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-pkzmyp.jpg
mathjax: true
---


# 循环依赖

Spring IoC 容器会在运行时检测到构造函数注入循环引用，并抛出 BeanCurrentlyInCreationException。
所以要避免构造函数注入，可以使用 setter 注入替代。

> @Autowired 是通过反射进行赋值。

例如：类 A 里面有类 B，类 B 里面有类 A

```JAVA

@Service
public class TestService1 {

    @Autowired
    TestService2 testService2;

    public String test1() {
        return "test1";
    }

}


@Service
public class TestService2 {

    @Autowired
    TestService1 testService1;

    public String test2() {
        return "test2";
    }

}


```

在 spring boot 2.6 之后默认是不支持循环依赖的

```LOG
Description:

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  testService1 (field top.sailliao.service.TestService2 top.sailliao.service.TestService1.testService2)
↑     ↓
|  testService2 (field top.sailliao.service.TestService1 top.sailliao.service.TestService2.testService1)
└─────┘


Action:

Relying upon circular references is discouraged and they are prohibited by default. 
Update your application to remove the dependency cycle between beans. 
As a last resort, it may be possible to break the cycle automatically by 
setting spring.main.allow-circular-references to true.


```

可以通过设置来修改

```YML
spring:
  main:
    allow-circular-references: true
```

我们例子的依赖属于**单例情况的filed属性的依赖**，对于**构造方法的循环依赖**和**非单利的循环依赖**还是会报错 BeanCurrentlyInCreationException

# 三级缓存

spring 内部有三级缓存

* singletonObjects : 一级缓存，用于保存实例化、注入、初始化完成的bean实例
* earlySingletonObjects : 二级缓存，用于保存实例化完成的bean实例，这个时候没设置属性
* singletonFactories : 三级缓存，用于保存bean创建工厂，以便于后面扩展有机会创建代理对象。

![](1.png)

## 源码

关于三级缓存的代码在 DefaultSingletonBeanRegistry 中

```JAVA
// 一级缓存
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

// 三级缓存
/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

// 二级缓存
/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

另外有比较重要的集合

```JAVA
// Spring容器会将每一个正在创建的Bean 标识符放在一个“当前创建Bean池”中，
// Bean标识符在创建过程中将一直保持在这个池中，而对于创建完毕的Bean将从当前创建Bean池中清除掉。
/** Names of beans that are currently in creation. */
private final Set<String> singletonsCurrentlyInCreation =
        Collections.newSetFromMap(new ConcurrentHashMap<>(16));

// 这个集合在

```

获取单例Bean的过程如下

```JAVA
@Override
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 一级缓存中获取
    Object singletonObject = this.singletonObjects.get(beanName);
    // 没有获取到
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 从二级缓存中获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            // 二级缓存中没获取到
            if (singletonObject == null && allowEarlyReference) {
                // 从三级缓存中获取 factory
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 创建出对象
                    singletonObject = singletonFactory.getObject();
                    // 添加到二级缓存中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 三级缓存中移除
                    // 加入到三级缓存中的前提是执行了构造器，所以构造器的注入的循环依赖没有办法解决
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}

```

二级缓存 earlySingletonObjects 中的元素只有这一个地方 **getSingleton** 能被添加。
在 removeSingleton、clearSingletonCache、addSingleton、addSingletonFactory 中移除


# Bean 的获取

在类 AbstractBeanFactory 中有很多重要的方法

```JAVA
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {

    protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
        throws BeansException {

        ...

        // 先去获取一次，如果不为null，此处就会走缓存了~~
        // Eagerly check singleton cache for manually registered singletons.
        Object sharedInstance = getSingleton(beanName);

        ...

        // 如果不是只检查类型，那就标记这个Bean被创建了~~添加到缓存里 也就是所谓的  当前创建Bean池
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        ...

        // Create bean instance.
        if (mbd.isSingleton()) {
        
            // 这个getSingleton方法不是SingletonBeanRegistry的接口方法  属于实现类DefaultSingletonBeanRegistry的一个public重载方法~~~
            // 它的特点是在执行singletonFactory.getObject();前后会执行beforeSingletonCreation(beanName);和afterSingletonCreation(beanName);  
            // 也就是保证这个Bean在创建过程中，放入正在创建的缓存池里  可以看到它实际创建bean调用的是我们的createBean方法~~~~
            sharedInstance = getSingleton(beanName, () -> {
                try {
                    return createBean(beanName, mbd, args);
                } catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }

        ...

    }

}
```

createBean 的实现在 AbstractAutowireCapableBeanFactory 中

```JAVA
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

        ...

        try {
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
            if (logger.isTraceEnabled()) {
                logger.trace("Finished creating instance of bean '" + beanName + "'");
            }
            return beanInstance;
        }

        ...

}


protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    throws BeanCreationException {

        ...

        // 创建Bean对象，并且将对象包裹在BeanWrapper 中
        instanceWrapper = createBeanInstance(beanName, mbd, args);

        ...

        // 再从Wrapper中把Bean原始对象（非代理~~~）生成出来, 这个时候这个Bean就有地址值了，就能被引用了~~~
        // 注意：此处是原始对象，这点非常的重要
        final Object bean = instanceWrapper.getWrappedInstance();


        // earlySingletonExposure 用于表示是否”提前暴露“原始对象的引用，用于解决循环依赖。
        // 对于单例Bean，该变量一般为 true   但你也可以通过属性allowCircularReferences = false来关闭循环引用
        // isSingletonCurrentlyInCreation(beanName) 表示当前bean必须在创建中才行

        // Eagerly cache singletons to be able to resolve circular references
        // even when triggered by lifecycle interfaces like BeanFactoryAware.
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isTraceEnabled()) {
                logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }
            // 上面讲过调用此方法放进一个ObjectFactory，二级缓存会对应删除的
            // getEarlyBeanReference的作用：调用SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()这个方法  否则啥都不做
            // 也就是给调用者个机会，自己去实现暴露这个bean的应用的逻辑~~~
            // 比如在getEarlyBeanReference()里可以实现AOP的逻辑~~~  参考自动代理创建器AbstractAutoProxyCreator  实现了这个方法来创建代理对象
            // 若不需要执行AOP的逻辑，直接返回Bean
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }

        // exposedObject 是最终返回的对象
        // Initialize the bean instance.
        Object exposedObject = bean; 

        ...

        // 填充属于，解决@Autowired依赖~
        populateBean(beanName, mbd, instanceWrapper);

        // 执行初始化回调方法们~~~
        exposedObject = initializeBean(beanName, exposedObject, mbd);
        
        // earlySingletonExposure：如果你的bean允许被早期暴露出去 也就是说可以被循环引用  那这里就会进行检查
        // 此段代码非常重要~~~~~但大多数人都忽略了它
        if (earlySingletonExposure) {
            // 此时一级缓存肯定还没数据，但是呢此时候二级缓存earlySingletonObjects也没数据
            // 注意，注意：第二参数为false  表示不会再去三级缓存里查了~~~
            // 此处非常巧妙的一点：：：因为上面各式各样的实例化、初始化的后置处理器都执行了，如果你在上面执行了这一句
            //  ((ConfigurableListableBeanFactory)this.beanFactory).registerSingleton(beanName, bean);
            // 那么此处得到的earlySingletonReference 的引用最终会是你手动放进去的Bean最终返回，完美的实现了"偷天换日" 特别适合中间件的设计
            // 我们知道，执行完此doCreateBean后执行addSingleton()  其实就是把自己再添加一次  **再一次强调，完美实现偷天换日**
            Object earlySingletonReference = getSingleton(beanName, false);
            if (earlySingletonReference != null) {
            
                // 这个意思是如果经过了initializeBean()后，exposedObject还是木有变，那就可以大胆放心的返回了
                // initializeBean会调用后置处理器，这个时候可以生成一个代理对象，那这个时候它哥俩就不会相等了 走else去判断吧
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } 

                // allowRawInjectionDespiteWrapping 这个值默认是false
                // hasDependentBean：若它有依赖的bean 那就需要继续校验了~~~(若没有依赖的 就放过它~)
                else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                    // 拿到它所依赖的Bean们~~~~ 下面会遍历一个一个的去看~~
                    String[] dependentBeans = getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                    
                    // 一个个检查它所以Bean
                    // removeSingletonIfCreatedForTypeCheckOnly 这个放见下面  在 AbstractBeanFactory 里面
                    // 简单的说，它如果判断到该 dependentBean 并没有在创建中的了的情况下,那就把它从所有缓存中移除~~~  并且返回true
                    // 否则（比如确实在创建中） 那就返回false 进入我们的if里面~  表示所谓的真正依赖
                    //（解释：就是真的需要依赖它先实例化，才能实例化自己的依赖）
                    for (String dependentBean : dependentBeans) {
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    // 若存在真正依赖，那就报错（不要等到内存移除你才报错，那是非常不友好的） 
                    // 这个异常是BeanCurrentlyInCreationException，报错日志也稍微留意一下，方便定位错误~~~~
                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName,
                                "Bean with name '" + beanName + "' has been injected into other beans [" +
                                StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                "] in its raw version as part of a circular reference, but has eventually been " +
                                "wrapped. This means that said other beans do not use the final version of the " +
                                "bean. This is often the result of over-eager type matching - consider using " +
                                "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }
        
        return exposedObject;
    }

    // 虽然是remove方法 但是它的返回值也非常重要
    // 该方法唯一调用的地方就是循环依赖的最后检查处~~~~~
    protected boolean removeSingletonIfCreatedForTypeCheckOnly(String beanName) {
        // 如果这个bean不在创建中  比如是ForTypeCheckOnly的  那就移除掉
        if (!this.alreadyCreated.contains(beanName)) {
            removeSingleton(beanName);
            return true;
        }
        else {
            return false;
        }
    }

}

```


# 流程总结

