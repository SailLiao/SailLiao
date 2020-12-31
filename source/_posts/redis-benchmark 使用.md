---
title: 'redis-benchmark 使用'
date: 2020-12-31 14:18:01
tags: Redis
cover: https://img9.51tietu.net/pic/2019-091404/4k54k4avjl44k54k4avjl4.jpg
---

Redis 自带了一个叫 redis-benchmark 的工具来模拟 N 个客户端同时发出 M 个请求。

```shell
Usage: redis-benchmark [-h <host>] [-p <port>] [-c <clients>] [-n <requests>] [-k <boolean>]

 -h <hostname>      Server hostname (default 127.0.0.1)
 -p <port>          Server port (default 6379)
 -s <socket>        Server socket (overrides host and port)
 -a <password>      Password for Redis Auth
 -c <clients>       Number of parallel connections (default 50)
 -n <requests>      Total number of requests (default 100000)
 -d <size>          Data size of SET/GET value in bytes (default 3)
 --dbnum <db>       SELECT the specified db number (default 0)
 -k <boolean>       1=keep alive 0=reconnect (default 1)
 -r <keyspacelen>   Use random keys for SET/GET/INCR, random values for SADD
  Using this option the benchmark will expand the string __rand_int__
  inside an argument with a 12 digits number in the specified range
  from 0 to keyspacelen-1. The substitution changes every time a command
  is executed. Default tests use this to hit random keys in the
  specified range.
 -P <numreq>        Pipeline <numreq> requests. Default 1 (no pipeline).
 -e                 If server replies with errors, show them on stdout.
                    (no more than 1 error per second is displayed)
 -q                 Quiet. Just show query/sec values
 --csv              Output in CSV format
 -l                 Loop. Run the tests forever
 -t <tests>         Only run the comma separated list of tests. The test
                    names are the same as the ones produced as output.
 -I                 Idle mode. Just open N idle connections and wait.

Examples:

 Run the benchmark with the default configuration against 127.0.0.1:6379:
   $ redis-benchmark

 Use 20 parallel clients, for a total of 100k requests, against 192.168.1.1:
   $ redis-benchmark -h 192.168.1.1 -p 6379 -n 100000 -c 20

 Fill 127.0.0.1:6379 with about 1 million keys only using the SET test:
   $ redis-benchmark -t set -n 1000000 -r 100000000

 Benchmark 127.0.0.1:6379 for a few commands producing CSV output:
   $ redis-benchmark -t ping,set,get -n 100000 --csv

 Benchmark a specific command line:
   $ redis-benchmark -r 10000 -n 10000 eval 'return redis.call("ping")' 0

 Fill a list with 10000 random elements:
   $ redis-benchmark -r 10000 -n 10000 lpush mylist __rand_int__

 On user specified command lines __rand_int__ is replaced with a random integer
 with a range of values selected by the -r option.
```

一般这样启动测试：

```bash

PS D:\Redis> .\redis-benchmark.exe -q -n 100000
PING_INLINE: 41425.02 requests per second
PING_BULK: 37091.99 requests per second
SET: 40683.48 requests per second
GET: 40322.58 requests per second
INCR: 38971.16 requests per second
LPUSH: 38022.81 requests per second
RPUSH: 38699.69 requests per second
LPOP: 39777.25 requests per second
RPOP: 40766.41 requests per second
SADD: 34855.35 requests per second
HSET: 39169.61 requests per second
SPOP: 41562.76 requests per second
LPUSH (needed to benchmark LRANGE): 41237.11 requests per second
LRANGE_100 (first 100 elements): 40666.94 requests per second
LRANGE_300 (first 300 elements): 41067.76 requests per second
LRANGE_500 (first 450 elements): 38789.76 requests per second
LRANGE_600 (first 600 elements): 39525.69 requests per second
MSET (10 keys): 40355.12 requests per second

```
这个工具使用起来非常方便，同时你可以使用自己的基准测试工具， 不过开始基准测试时候，我们需要注意一些细节。

## 只运行一些测试用例的子集

你不必每次都运行 redis-benchmark 默认的所有测试。 使用 -t 参数可以选择你需要运行的测试用例

```bash
./redis-benchmark.exe -t set,lpush -n 100000 -q
SET: 42662.11 requests per second
LPUSH: 42408.82 requests per second
```

## 选择测试键的范围大小

假设我们想设置 10 万随机 key 连续 SET 100 万次，我们可以使用下列的命令：

```bash
./redis-benchmark.exe -t set -r 100000 -n 1000000
====== SET ======
  1000000 requests completed in 23.62 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

96.15% <= 1 milliseconds
99.93% <= 2 milliseconds
100.00% <= 3 milliseconds
100.00% <= 4 milliseconds
100.00% <= 4 milliseconds
42342.38 requests per second


# 清除全部
redis-cli flushall

```

## 使用 pipelining

默认情况下，每个客户端都是在一个请求完成之后才发送下一个请求 （benchmark 会模拟 50 个客户端除非使用 -c 指定特别的数量）

这意味着服务器几乎是按顺序读取每个客户端的命令。

记得在多条命令需要处理时候使用 pipelining。

## 陷阱和错误的认识

第一点是显而易见的：基准测试的黄金准则是使用相同的标准。 用相同的任务量测试不同版本的 Redis，或者用相同的参数测试测试不同版本 Redis。 如果把 Redis 和其他工具测试，那就需要小心功能细节差异。

* Redis 是一个服务器：所有的命令都包含网络或 IPC 消耗。这意味着和它和 SQLite， Berkeley DB， Tokyo/Kyoto Cabinet 等比较起来无意义， 因为大部分的消耗都在网络协议上面。
* Redis 的大部分常用命令都有确认返回。有些数据存储系统则没有（比如 MongoDB 的写操作没有返回确认）。把 Redis 和其他单向调用命令存储系统比较意义不大。
* 简单的循环操作 Redis 其实不是对 Redis 进行基准测试，而是测试你的网络（或者 IPC）延迟。想要真正测试 Redis，需要使用多个连接（比如 redis-benchmark)， 或者使用 pipelining 来聚合多个命令，另外还可以采用多线程或多进程。
* Redis 是一个内存数据库，同时提供一些可选的持久化功能。 如果你想和一个持久化服务器（MySQL, PostgreSQL 等等） 对比的话， 那你需要考虑启用 AOF 和适当的 fsync 策略。
* Redis 是单线程服务。它并没有设计为多 CPU 进行优化。如果想要从多核获取好处， 那就考虑启用多个实例吧。将单实例 Redis 和多线程数据库对比是不公平的。

## 影响 Redis 性能的因素

* 网络带宽和延迟通常是最大短板。
* CPU 是另外一个重要的影响因素，由于是单线程模型，Redis 更喜欢大缓存快速 CPU， 而不是多核。
* Redis 在 VM 上会变慢。
* 当使用网络连接时，并且以太网网数据包在 1500 bytes 以下时， 将多条命令包装成 pipelining 可以大大提高效率。事实上，处理 10 bytes，100 bytes， 1000 bytes 的请求时候，吞吐量是差不多的
* 在高配置下面，客户端的连接数也是一个重要的因素。得益于 epoll/kqueue， Redis 的事件循环具有相当可扩展性。Redis 已经在超过 60000 连接下面基准测试过， 仍然可以维持 50000 q/s。一条经验法则是，30000 的连接数只有 100 连接的一半吞吐量。
* 在高配置下面，可以通过调优 NIC 来获得更高性能。
* 在不同平台下面，Redis 可以被编译成不同的内存分配方式（libc malloc, jemalloc, tcmalloc），他们在不同速度、连续和非连续片段下会有不一样的表现。


## 总结

文章来源：[Redis有多快?](http://www.redis.cn/topics/benchmarks.html)