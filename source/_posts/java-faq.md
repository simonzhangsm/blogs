---
title: java-faq
date: 2016-08-12 10:33:59
tags: java
---

## 0. TreeMap LinkedHashMap的实现原理 如何保证有序
容器有序指迭代容器时按顺序返回。
TreeMap使用红黑树实现，基于比较实现有序。
LinkedHashMap使用双向链表保证按插入顺序存储。
参着<http://yikun.github.io/2015/04/02/Java-LinkedHashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/>

## 1. java.util.concurrent包里有什么？CyclicBarrier原理是什么？

java.util.concurrent包括

- atomic操作类
- locks 参考 <http://yhjhappy234.blog.163.com/blog/static/3163283220135491412111/>
- concurrent容器类（Queue、Deque、NavigableMap、HashMap、SkipListMap，CopyOnWriteArrayList/CopyOnWriteArraySet）
- 同步原语（CyclicBarrier、CountDownLatch、Semaphore）
- 异步任务支持（RecursiveAction、RecursiveTask、ForkJoinTask、ForkJoinPool、ThreadPoolExecutor、ExecutorService、Executors、ScheduledThreadPoolExecutor）
- Future支持（CompletableFuture、ScheduledFuture）

CyclicBarrier构造的时候设置需要等待的线程数counter，让各线程调用await来等待Condition并将该counter减小，当减小到0时，将给该Condition发信号，唤醒所有的等待线程。内部状态的由ReentrantLock保护，关键在于Conditon和该ReentrantLock共用一个Condition，这样就可以在wait该Condition时将外部所释放，让其它线程有机会获得锁。


## 2. java.nio里的主要包括那些内容

## 3. 虚拟机GC的有哪些回收器，策略是什么

## 4. 有哪些ClassLoader，什么是双亲分派，用来解决什么问题

> http://my.oschina.net/xionghui/blog/499725

- BootstrapClassLoader 用来引导程序启动，不继承ClassLoader
- ExtClassLoader 用来加载java的扩展库
- AppClassLoader 用来加载应用类

他们组成树状结构

双亲分派机制

- 当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。
- 当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。
- 如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；
- 若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。

## 5. 内存模型是怎么设计的，volatile有什么作用
