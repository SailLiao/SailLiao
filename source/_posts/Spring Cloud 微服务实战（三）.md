---
title: Spring Cloud 微服务实战（三）
date: 2021-04-12 16:49:56
tags: Spring Cloud
cover: https://img10.51tietu.net/pic/20191029/tgrhvdbw3wjtgrhvdbw3wj.jpg
---

书接上回

# Hystrix

服务拆分后，如果其中某一个服务调用过慢，会导致调用方线程因为等待服务响应而挂起知道调用失败，在高并发的情况下，这些挂起的服务，会让新的请求被阻塞从而导致整个服务不可用。
**断路器** 能很好的解决这一情况

![](https://sailliao.oss-cn-beijing.aliyuncs.com/img/spring-cloud-3-1.png)

在 2020-12-22 日 spring cloud 移除了 zuul Hystrix 等组件，原因是 2018 年开始 Netflix 公司就不再对 Hystrix、Ribbon、Zuul、Eureka 等进行新特性开发。替代可以使用 Spring Cloud Alibaba

## 原理分析

先看一下**命令模式**

将来自客户端的请求封装成一个对象， 从而让你可以使用不同的请求对客户端进行参数化。 它可以被用于实现 “ 行为请求者 ” 与 “ 行为实现者 ” 的解耦， 以便使两者可以适应变化。

```java

// 接收者
public class Receiver {
    public void action() {
        // 真正的业务逻辑
    }
}

// 抽象命令
public interface Command {
    void execute();
}

// 具体命令实现
public class ConcreteCommand implements Command{

    private Receiver receiver;

    public ConcreteCommand (Receiver receiver) {
        this.receiver = receiver;
    }

    @Override
    public void execute() {
        this.receiver.action();
    }
}

// 客户端调用者
public class Invoker {

    private Command command;

    public void setCommand(Command command) {
        this.command = command;
    }

    public void action(){
        this.command.execute();
    }

}


public class Client {

    public static void main(String[] args) {

        Receiver receiver = new Receiver();
        Command command = new ConcreteCommand(receiver);

        Invoker invoker = new Invoker();
        invoker.setCommand(command);
        invoker.action(); // 客户端通过调用者来执行命令

    }

}

```

调用者 Invoker 与操作者 Receiver 通过 Command 命令接口实现了解耦。对于调用者来说， 我们可以为其注入多个命令操作， 比如新建文件、复制文件、 删除文件这样三个操作，调用者只需在需要的时候直接调用即可，而不需要知道这些操作命令实际是如何实现的。

* HystrixCommand: 用在依赖的服务返回单个操作结果的时候。
* HystrixObservableCommand: 用在依赖的服务返回多个操作结果的时候。

## 2. 命令执行

HystrixComrnand 实现了下面两个执行方式

* execute()

同步执行，从依赖的服务 返回一个单一的结果对象， 或是在发生错误的时候抛出异常

* queue()

异步执行， 直接返回 一个Future对象，其中包含了服务执行 结束时要返回的单一结果对象

```java
R value = command.execute() ;
Future<R> fValue = command.queue();
```
HystrixObservableCommand 实现了另外两种执行方式

* observe()

返回 Observable 对象，它代表了操作的多个结果，它是 一个Hot Observable

* toObservable()

同样会返回Observable对象， 也代表了操作的多个结果，但它返回的是 一个Cold Observable。

Hystrix 底层大量使用了 RxJava

### Rxjava

RxJava本质上是一个异步操作库，是一个能让你用极其简洁的逻辑去处理繁琐复杂任务的异步事件库。

上代码

假设我们安居客用户App上有个需求，需要从服务端拉取上海浦东新区塘桥板块的所有小区Community[] communities，每个小区下包含多套房源List<House> houses；我们需要把塘桥板块的所有总价大于500W的房源都展示在App的房源列表页。用于从服务端拉取communities需要发起网络请求，比较耗时，因此需要在后台运行。而这些房源信息需要展示到App的页面上，因此需要在UI线程上执行。

普通的方式

```java
new Thread() {
    @Override
    public void run() {
        super.run();
        //从服务端获取小区列表
        List<Community> communities = getCommunitiesFromServer();
        for (Community community : communities) {
            List<House> houses = community.houses;
            for (House house : houses) {
                if (house.price >= 5000000) {
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            //将房子的信息添加到屏幕上
                            addHouseInformationToScreen(house);
                        }
                    });
                }
            }
        }
    }
}.start();
```

使用RxJava的写法是这样的：

```java
Observable.from(getCommunitiesFromServer())
    .flatMap(new Func1<Community, Observable<House>>() {
        @Override
        public Observable<House> call(Community community) {
            return Observable.from(community.houses);
        }
    }).filter(new Func1<House, Boolean>() {
        @Override
        public Boolean call(House house) {
            return house.price>=5000000;
        }
    }).subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<House>() {
        @Override
        public void call(House house) {
            //将房子的信息添加到屏幕上
            addHouseInformationToScreen(house);
        }
    });
```
再配合 Lambda

```java
Observable.from(getCommunitiesFromServer())
        .flatMap(community -> Observable.from(community.houses))
        .filter(house -> house.price>=5000000).subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(this::addHouseInformationToScreen);
```

### RxJava 的观察订阅模式

**Observable** 对象就是RxJava中的核心内容之一，可以把它理解为 “事件源” 或是 “被观察者”， 与其对应的 **Subscriber** 对象，可以理解为 “订阅者” 或是  观察者”。 这两个对象是RxJava响应式编程的重要组成部分。

1. Observable（事件源，被观察者） 用来向 Subscriber（订阅者，观察者）发布事件，Subscriber 则在接收到事件后对其进行处理，这里的事件通常是对一来服务的调用。
2. 一个 Observable 可以发出多个事件，直到结束或者发生异常。
3. Observable 对象每发出一个事件，就会调用观察者 Subscriber 对象的 onNext() 方法。
4. 每一个Observable的执行，最后 一定会通过调用 Subscriber.onCompleted() 或者 Subscriber.onError() 来结束该事件的操作流。

Observable 有两个不同的概念， Hot Observalbe 、Cold Observable 分别对应了 command.observe() 和 command.toObservable() 的返回对象，在 execute() 和 queue() 中也使用了 RxJava。

* execute() 是通过 queue() 返回的异步对象 Future<R> 的 get() 方法来实现同步执行的。 该方法会等待任务执行结束， 然后获得 R 类型的结果进行返回。
* queue() 则是通过 toObservable() 来获得一个 Cold Observable, 并且通过 toBlocking() 将该 Observable 转换成 BlockingObservable, 它可以把数据以阻塞的方式发射出来。 而 toFuture 方法则是 把 BlockingObservable 转换为一个 Future , 该方法只是创建一个 Future 返回并不会阻塞， 这使得消费者可以自己决定如何处理异步操作。 而execute() 就是直接使用了 queue() 返回的 Future 中的阻塞方法 get() 来实现同步操作的。 同时通过这种方式转换的 Future 要求 Observable 只发射一个数据，所以这两个实现都只能返回单一结果。

## 3. 结果是否被缓存

若当前命令的请求缓存功能是被启用的， 并且该命令缓存命中， 那么缓存的结果会立即以 Observable 对象的形式返回

## 4. 断路器是否打开

在命令结果没有命中缓存的时候，Hystrix 需要检查断路器是否打开，如果打开，那么 Hystrix 不会执行命令，而是转到 fallback 处理逻辑（步骤8），如果是打开的，就进行 5 来检查是否有资源执行命令

## 5. 线程池/请求队列/信号量是否占满

如果占满，也不会执行，会执行 fallback 

## 6. HystrixObservableCommand.construct() 或 HystrixCommand.run()

Hystrix会根据我们编写的方法来决定采取什么样的方式去请求依赖服务。

* HystrixCommand.run(): 返回一个单一 的结果，或者抛出异常。
* HystrixObservableCommand.construct(): 返回一个Observable对象来发射多个结果，或通过onError发送错误通知。

如果超时，会抛出 TimeoutException

## 7. 计算断路器的健康值

Hystrix会将 “ 成功 ”、 “ 失败 ”、 “ 拒绝 ”、 “ 超时 ” 等信息报告给断路器， 而断路器会维护一组计数器来统计这些数据。
断路器会使用这些统计数据来决定是否要将断路器打开， 来对某个依赖服务的请求进行 “ 熔断／短路 ”，直到恢复期结束。 若在恢复期结束后， 根据统计数据判断如果还是未达到健康指标，就再次 “ 熔断／短路 ”。

## 8. fallback 处理(降级)

命令执行失败的时候，Hystrix 会进入 fallback 尝试回退处理，也被称为 **服务降级**， 有以下几种情况

* 第4步， 当前命令处于 “ 熔断I短路 ” 状态， 断路器是打开的时候。
* 第5步， 当前命令的线程池、 请求队列或 者信号量被占满的时候。
* 第6步，HystrixObservableCommand.construct() 或 HystrixCommand.run() 抛出异常的时候。

## 9. 返回成功的响应

![](https://sailliao.oss-cn-beijing.aliyuncs.com/img/spring-cloud-3-2.png)


# 断路器原理

那么断路器是如何决策熔断和记录信息的呢，我们先看看 HystrixCircuitBreaker 的定义

```java
public interface HystrixCircuitBreaker {

    // 每个 Hystrix 命令的请求都通过它判断是否被执行。
    public boolean allowRequest();

    // 返回当前断路器是否打开。
    public boolean isOpen();

    // 用来闭合断路器
    void markSuccess();

    public static class Factory {}

    static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {}

    static class NoOpCircuitBreaker implements HystrixCircuitBreaker {}
}
```

# 依赖隔离

“舱壁模式” Hystrix 则使用该模式实现线程池的隔离，它会为每一个依赖服务创建一个独立的线程池， 这样就算某个依赖服务出现延迟过高的情况， 也只是对该依赖服务的调用产生影响， 而不会拖慢其他的依赖服务。

在 Hystrix 中除了可使用线程池之外， 还可以使用信号量来控制单个依赖服务的并发度， 信号量的开销远比线程池的开销小， 但是它不能设置超时和实现异步访问。 所以， 只有在依 赖 服 务是足够可靠的情况 下才使用信号量。

* 命令执行：如果将隔离策略参数 execution.isolation.strategy 设置为 SEMAPHORE, Hystrix 会使用信号量替代线程池来控制依赖服务的并发
* 降级逻辑：当 Hystrix 尝试降级逻辑时， 它会在调用线程中使用信号量。



