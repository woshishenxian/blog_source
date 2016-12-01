---
title: Rxjava学习：Rxjava的进阶
date: 2016-09-05 15:56:47
tags: Rxjava
categories: Rxjava学习
---
### 前言
最近有点乱，项目忙着上线，临近这个时候，项目总会出现各种问题，测试出各种bug，有的没得一大堆，尤其是出现memory leak问题，关于memory leak一直想写篇博客谈谈我的想法，这里不多说，咱来接着聊一聊Rxjava的学习。
<!-- more-->

![Rxjava学习：Rxjava的进阶](/imgs/4c1c8e5bfd6663f788acb9a9665f3ffc.jpg)

### 线程控制 —— Scheduler
线程的控制也是Rxjava种很重要的一块，Rxjava在设计上，针对数据操作的性质作了考虑，由于不同的操作所需要的线程要求不同，这样就需要 Scheduler（调度器）。

默认情况下的Rxjava是不会对线程进行变换的，也就是说，事件发生在哪个线程，最终就会运行在哪个线程。

先来看看Scheduler类 API中提供的线程选择：
- Schedulers.immediate(): 默认的 Scheduler，在当前线程立即开始执行任务。

- Schedulers.newThread(): 为每个任务创建一个新线程。

- Schedulers.io(): 。用于IO密集型任务，如对文件的读写、数据库的操作、网络的访问等，这个调度器的线程池会根据需要增长；对于普通的计算任务，使用Schedulers.computation()；Schedulers.io( )默认是一个CachedThreadScheduler，很像一个有线程缓存的新线程调度器。

- Schedulers.computation(): 计算所使用的 Scheduler。如事件循环或和回调处理，不要用于IO操作(IO操作请使用Schedulers.io())；默认线程数等于处理器的数量。

- 另外， 对于Android 还提供一个专用的 AndroidSchedulers.mainThread()，它指定的操作将在 Android 主线程运行。

对于android来说，有了以上几个基本够用了，Rxjava提供了subscribeOn() 和 observeOn() 这两个方法来对线程进行控制。
- subscribeOn() ：用来指定subscribe()方法发生的线程，通常 只需调用一次,位置则随意。
- observeOn()：用来指定其接下来的方法的回调发生的线程。
举个例子解释一下：
```
Observable.just(1, 2, 3, 4) // IO 线程，由 subscribeOn() 指定
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.newThread())
    .map(mapOperator) // 新线程，由 observeOn() 指定
    .observeOn(Schedulers.io())
    .map(mapOperator2) // IO 线程，由 observeOn() 指定
    .observeOn(AndroidSchedulers.mainThread) 
    .subscribe(subscriber);  // Android 主线程，由 observeOn() 指定
```
由 subscribeOn(Schedulers.io()) 指定 subscribe() 最终将发生在 io线程中，而observeOn(AndroidSchedulers.mainThread) 则表示 subscriber 的方法将发生在 主线程 中。 在来看一下，observeOn(Schedulers.newThread()) 在表示下面的方法 map(mapOperator) 中，mapOperator任务将 发生在一个新线程中，mapOperator2 发生在io线程中。
