---
title: Fork/Join框架
date: 2021-08-02 15:18:32
tags: Java
cover: https://img20.51tietu.net/pic/2016-081006/20160810060908gld1ogsh1yq.jpg
---

# 什么是 Fork/Join

Fork/Join 框架是 Java7 提供了的一个用于并行执行任务的框架， 是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。

## 分治

我们再通过 Fork 和 Join 这两个单词来理解下 Fork/Join 框架，Fork 就是把一个大任务切分为若干子任务并行的执行，Join 就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算 1+2+。。＋10000，可以分割成 10 个子任务，每个子任务分别对 1000 个数进行求和，最终汇总这 10 个子任务的结果。

是不是和mapreduce有点像

## 工作窃取算法

工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行。

### 为什么需要使用工作窃取算法

假如我们需要做一个比较大的任务，我们可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，于是把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应，比如 A 线程负责处理 A 队列里的任务。但是有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用**双端队列**，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

工作窃取算法的优点是充分利用线程进行并行计算，并减少了线程间的竞争，其缺点是在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且消耗了更多的系统资源，比如创建多个线程和多个双端队列。

# 原理

在了解原理前，其实可以思考下，自己来设计个 fork/join 框架应该怎么设计。

1. 因为需要切分成很多的小任务，所以任务之间不能有依赖，独立的执行功能
2. 事先分配好固定的双端队列
3. 每个线程执行对应的队列的任务
4. 当前线程做完队列的任务后，查看其他队列有没有多余的任务，有的话，从头取出来执行
5. 执行完的任务，结果放到一个队列里面，启动线程从队列里面拿数据，然后合并这些数据

# 实现

## ForkJoinTask

![](2.png)

我们要使用 ForkJoin 框架，必须首先创建一个 ForkJoin 任务。它提供在任务中执行 fork() 和 join() 操作的机制，通常情况下我们不需要直接继承 ForkJoinTask 类，而只需要继承它的子类，Fork/Join 框架提供了以下两个子类：

* RecursiveAction：用于没有返回结果的任务。
* RecursiveTask ：用于有返回结果的任务。

## ForkJoinPool

![](1.png)

ForkJoinTask 需要通过 ForkJoinPool 来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

构造函数

```java
public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode)
```

* parallelism

可并行级别，Fork/Join框架将依据这个并行级别的设定，决定框架内并行执行的线程数量。并行的每一个任务都会有一个线程进行处理，但是千万不要将这个属性理解成Fork/Join框架中最多存在的线程数量，也不要将这个属性和ThreadPoolExecutor线程池中的corePoolSize、maximumPoolSize属性进行比较，因为ForkJoinPool的组织结构和工作方式与后者完全不一样。而后续的讨论中，读者还可以发现Fork/Join框架中可存在的线程数量和这个参数值的关系并不是绝对的关联（有依据但并不全由它决定）。

* factory

当Fork/Join框架创建一个新的线程时，同样会用到线程创建工厂。只不过这个线程工厂不再需要实现ThreadFactory接口，而是需要实现ForkJoinWorkerThreadFactory接口。后者是一个函数式接口，只需要实现一个名叫newThread的方法。在Fork/Join框架中有一个默认的ForkJoinWorkerThreadFactory接口实现：DefaultForkJoinWorkerThreadFactory。

* handler

异常捕获处理器。当执行的任务中出现异常，并从任务中被抛出时，就会被handler捕获。

* asyncMode

这个参数也非常重要，从字面意思来看是指的异步模式，它并不是说Fork/Join框架是采用同步模式还是采用异步模式工作。Fork/Join框架中为每一个独立工作的线程准备了对应的待执行任务队列，这个任务队列是使用数组进行组合的双向队列。即是说存在于队列中的待执行任务，**即可以使用先进先出的工作模式，也可以使用后进先出的工作模式。**

当asyncMode设置为ture的时候，队列采用先进先出方式工作；反之则是采用后进先出的方式工作，该值默认为false

ForkJoinPool还有另外两个构造函数，一个构造函数只带有parallelism参数，既是可以设定Fork/Join框架的最大并行任务数量；另一个构造函数则不带有任何参数，对于最大并行任务数量也只是一个默认值——当前操作系统可以使用的CPU内核数量**Runtime.getRuntime().availableProcessors()**。实际上ForkJoinPool还有一个私有的、原生构造函数，之上提到的三个构造函数都是对这个私有的、原生构造函数的调用。

```java
private ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        int mode,
                        String workerNamePrefix) {
    this.workerNamePrefix = workerNamePrefix;
    this.factory = factory;
    this.ueh = handler;
    this.config = (parallelism & SMASK) | mode;
    long np = (long)(-parallelism); // offset ctl counts
    this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
}
```

如果你对Fork/Join框架没有特定的执行要求，可以直接使用不带有任何参数的构造函数。也就是说推荐基于当前操作系统可以使用的CPU内核数作为Fork/Join框架内最大并行任务数量，这样可以保证CPU在处理并行任务时，尽量少发生任务线程间的运行状态切换（实际上单个CPU内核上的线程间状态切换基本上无法避免，因为操作系统同时运行多个线程和多个进程）。


## ForkJoinWorkerThread

执行任务的线程

![](3.png)

# 使用

例如我们现在要计算从 1 加到 100

使用 Fork／Join 框架首先要考虑到的是如何分割任务，如果我们希望每个子任务最多执行10个数的相加，那么我们设置分割的阈值是10

```java
public class CountTask extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 10;
    private int first;
    private int last;

    public CountTask(int first, int last) {
        this.first = first;
        this.last = last;
    }


    @Override
    protected Integer compute() {
        int sum = 0;
        if (last - first <= THRESHOLD) {
            for (int i = first; i <= last; i++) {
                sum += i;
            }
        } else {
            int middle = (first + last) / 2;
            CountTask leftTask = new CountTask(first, middle);
            CountTask rightTask = new CountTask(middle + 1, last);
            leftTask.fork();
            rightTask.fork();
            sum = leftTask.join() + rightTask.join();

            /*int leftResult = leftTask.join();
            int rightResult = rightTask.join();
            sum = leftResult + rightResult;*/

        }
        return sum;
    }


    public static void main(String[] args) throws Exception {

        CountTask countTask = new CountTask(1, 100);
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        Future result = forkJoinPool.submit(countTask);
        System.out.println(result.get());
    }
}
```

## 流程分析

### fork()

```java
public final ForkJoinTask<V> fork() {
    Thread t;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        ForkJoinPool.common.externalPush(this);
    return this;
}
```
fork方法用于将新创建的子任务放入当前线程的work queue队列中，Fork/Join框架将根据当前正在并发执行ForkJoinTask任务的ForkJoinWorkerThread线程状态，决定是让这个任务在队列中等待，还是创建一个新的ForkJoinWorkerThread线程运行它，又或者是唤起其它正在等待任务的ForkJoinWorkerThread线程运行它。

这里面有几个元素概念需要注意，ForkJoinTask任务是一种能在Fork/Join框架中运行的特定任务，也只有这种类型的任务可以在Fork/Join框架中被拆分运行和合并运行。ForkJoinWorkerThread线程是一种在Fork/Join框架中运行的特性线程，它除了具有普通线程的特性外，最主要的特点是**每一个ForkJoinWorkerThread线程都具有一个独立的任务等待队列（work queue）**，这个任务队列用于存储在本线程中被拆分的若干子任务。

ForkJoinWorkerThread 里的 workQueue 是这样的

```java
final ForkJoinPool.WorkQueue workQueue; // work-stealing mechanics
```

pushTask方法把当前任务存放在ForkJoinTask数组队列里。然后再调用ForkJoinPool的signalWork()方法唤醒或创建一个工作线程来执行任务。代码如下：

```java
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)
            growArray();
    }
}
```

### join()

join方法用于让当前线程阻塞，直到对应的子任务完成运行并返回执行结果。或者，如果这个子任务存在于当前线程的任务等待队列（work queue）中，则取出这个子任务进行“递归”执行。其目的是尽快得到当前子任务的运行结果，然后继续执行。

```java
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        reportException(s);
    return getRawResult();
}
```

它首先调用doJoin方法，通过doJoin()方法得到当前任务的状态来判断返回什么结果，任务状态有4种：
1. 已完成（NORMAL）
2. 被取消（CANCELLED）
3. 信号（SIGNAL）
4. 出现异常（EXCEPTIONAL）。

如果任务状态是已完成，则直接返回任务结果。
如果任务状态是被取消，则直接抛出CancellationException
如果任务状态是抛出异常，则直接抛出对应的异常
让我们分析一下doJoin方法的实现

```java
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    return (s = status) < 0 ? s :
        ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        wt.pool.awaitJoin(w, this, 0L) :
        externalAwaitDone();
}

final int doExec() {
    int s; boolean completed;
    if ((s = status) >= 0) {
        try {
            completed = exec();
        } catch (Throwable rex) {
            return setExceptionalCompletion(rex);
        }
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}
```

在doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完成，如果执行完成，则直接返回任务状态；如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成，则设置任务状态为NORMAL，如果出现异常，则记录异常，并将任务状态设置为EXCEPTIONAL。

## 异常处理

ForkJoinTask在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以ForkJoinTask提供了**isCompletedAbnormally()**方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的getException方法获取异常。使用如下代码：

```java
if(task.isCompletedAbnormally()){
    System.out.println(task.getException());
}
```
getException方法返回Throwable对象，如果任务被取消了则返回CancellationException。如果任务没有完成或者没有抛出异常则返回null。

```java
public final Throwable getException() {
    int s = status & DONE_MASK;
    return ((s >= NORMAL)    ? null :
            (s == CANCELLED) ? new CancellationException() :
            getThrowableException());
}
```
