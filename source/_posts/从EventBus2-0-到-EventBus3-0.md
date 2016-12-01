---
title: 从EventBus2.0 到 EventBus3.0
date: 2016-08-28 14:37:35
tags: [EventBus2.0,EventBus3.0]
categories: EventBus
---
![配图](/imgs/EventBus-Top.jpg)


### 前言：
以前开发一直在用 EventBus2.0，最近想学点新东西，看到EventBus竟然出3.0了，看时间是早就有了，但一直没接触过，就学习学习，也顺便谈谈我在使用两个版本时，对它们的不同的感受。
开讲之前先付一下源码地址：https://github.com/greenrobot/EventBus
<!-- more-->
### 介绍一下：
EventBus是由大名鼎鼎的greenrobot出品的一个用于Android中事件发布/订阅的库。简单点说就是用于Fragment，Activity，Service，线程之间进行数据传递，它为开发者提供除了 intent、handler、boardcast这几种传递数据的方式之外的一种选择，其优点在于 几乎不怎么消化资源，并且代码优雅简洁。

### EventBus 的组成

![EventBus-Publish-Subscribe](/imgs/EventBus-Publish-Subscribe.png)

从上图我们可以看到，EventBus 作为事件总线，有3个重要做成部分：
- Publisher: 发布者。 表示数据的持有者，通过eventbus.post(obj)方法将数据传递出去，然而并不关心数据是否被接受，以及数据的传递过程。获取方法也很简单：
```
EventBus.getDefault();
``` 
- Event:事件。这里我习惯称之为数据，就是你所要传递出得对象。
- Subscriber: 订阅者。 或者好理解点的话，可是称之为接收者，数据将通过这些函数来进行接受。EventBus这里提供了四个函数且只能用这四个：**onEvent，onEventMainThread，onEventBackgroundThread，onEventAsync .**当然，上面的方法是对于EventBus2.0中用到的 下面会重点讲一下3.0的用法。

### EventBus 四个Subscriber

首先在将四种方式之前不得不说一下ThreadMode, ThreadMode 是EventBus中一个很重要的概念，其本身是一个enum，他同样提供了四个默认属性值：Async，BackgroundThread，MainThread，PostThread；而在3.0中则改为
ASYNC，BACKGROUND，MAIN，POSTING。分别对应四个方法。

先来说一下2.0中的Subscriber
>**onEvent**
对应 PostThread，当使用onEvent作为订阅函数时，发布者在哪个线程发布事件，onEvent就会在**哪个线程**接收事件。因为函数执行的线程不确定，这就要求用户 最好不要执行耗时操作，可能会出现线程阻塞或者事件分发不及时的问题。
注意一点：**方法的修饰符 最好 使用 public **

```
public void onEvent（Object obj）{
  
}
```
>** onEventMainThread**
对应 MainThread，当使用onEventMainThread作为订阅函数时，发布者不管在哪个线程发布事件，onEventMainThread都会在**主线程**接收事件。这就要求用户 一定不要执行耗时操作，不然会造成线程阻塞。

```
public void onEventMainThread（Object obj）{
  
}
```

>** onEventBackground**
对应 BackgroundThread，当使用onEventBackground作为订阅函数时，发布者如果在主线程发布事件，onEventBackground将会**新开一个子线程**并接收事件，但是如果是在子线程中发布事件，onEventBackground将会直接在**该子线程**中接受事件。

```
public void onEventBackground（Object obj）{
  
}
```

>** onEventAsync**
对应 Async，当使用onEventAsync作为订阅函数时，发布者无论在哪个线程发布事件，onEventAsync都会**新开一个子线程**并接收事件。

```
public void onEventAsync（Object obj）{
  
}
```

**而相对来说，3.0 改变对需要固定函数名的设定，提供了一种新的形式，改用注解@Subscribe ，在使用中更加灵活，而且 这里 ThreadMode 将被直接使用。**
> onEvent 函数的实现

```
//发布线程中，调用
@Subscribe(threadMode = ThreadMode.POSTING)
public void eventbusPosting(Object object){
}
```
> onEventMainThread 函数的实现

```
//主线程中，调用
@Subscribe(threadMode = ThreadMode.MAIN)
public void eventbusMain(Object object){
}
```
> onEventBackground 函数的实现

```
//发布线程为主线程，新开子线程，调用
//发布线程为子线程，该线程调用
@Subscribe(threadMode = ThreadMode.BACKGROUND)
public void eventbusBackground(Object object){
}
```
> onEventAsync 函数的实现

```
//新开子线程调用
@Subscribe(threadMode = ThreadMode.ASYNC)
public void eventbusAsync(Object object){
}
```
**最后说一点：EventBus 并没有摒弃固定函数名的形式，开发者仍旧可以使用这种形式，不过在使用过程中必须要 添加注解 @ Subscribe 但不用指定ThreadMode**

### EventBus的基本使用

#### 引入
```
compile 'org.greenrobot:eventbus:3.0.0'
```
#### 定义事件
```
public class MessageEvent { /* Additional fields if needed */ }
```
#### 注册订阅者
```
EventBus.getDefault().register(this);
```
#### 发送事件
```
EventBus.getDefault().post(messageEvent);
```
#### 接收事件
```
//eventbus2.0形式
public void onEvent(MessageEvent event){

}
或者
//eventbus3.0形式
@Subscribe(threadMode = ThreadMode.POSTING)
public void postingME(MessageEvent event){

}
```
#### 注销订阅
```
EventBus.getDefault().unregister(this);
```
