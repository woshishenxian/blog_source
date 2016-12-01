---
title: Rxjava学习：谈谈Rxjava的使用
date: 2016-08-25 17:27:24
categories: Rxjava学习
tags:
---
![配图](/imgs/8c91ad2096d7aaea9dc52dc4c86a8d58.jpg)

### 前言

在 [Rxjava学习:初识Rxjava](http://www.jianshu.com/p/264a1322669a) 一文中，我们对Rxjava的几个基本概念做了介绍，今天就来谈谈对Rxjava的使用。
<!-- more-->
### Rxjava的具体使用
下面还是从几个基本概念开始入手：
#### 创建Observer
作为Rxjava处理数据的最终出口，首先我们需要对其进行实现。

```
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```
除了 Observer 接口之外，RxJava 还提供了一个实现了 Observer 的抽象类：Subscriber。实际上在Observer接口的使用过程中，也是将其转换成Subscriber 来使用。
 下面我们来看一下订阅方法的源码实现：
```
public final Subscription subscribe(final Observer<? super T> observer) {
    if (observer instanceof Subscriber) {
        return subscribe((Subscriber<? super T>)observer);
    }
    if (observer == null) {
        throw new NullPointerException("observer is null");
    }
    return subscribe(new ObserverSubscriber<T>(observer));
}
```
可以看到 如果是 **Subscriber** 将直接被使用，如果是**observer**将被转换为 **ObserverSubscriber** 来使用。这里提到了** ObserverSubscriber **类 我们再来看一下 其源码，就会很清晰的理解这个过程了：
```
public final class ObserverSubscriber<T> extends Subscriber<T> {
    final Observer<? super T> observer;
    public ObserverSubscriber(Observer<? super T> observer) {
        this.observer = observer;
    }
        @Override
    public void onNext(T t) {
        observer.onNext(t);
    }
        @Override
    public void onError(Throwable e) {
        observer.onError(e);
    }
        @Override
    public void onCompleted() {
        observer.onCompleted();
    }
}
```

当然Subscriber 对 Observer 接口也进行了一些扩展。 下面提一下 Subscriber 与 Observer 使用上的不同：
- onStart(): Subscriber 对外提供了onStart 方法用于实现，onStart 在Subscriber中是一个空方法，用户可以选择性实现。**会在整个订阅过程开始之前被调用，与Subscribe()方法发生在同一线程，可以做一些 比如：数据的初始化 之类的操作，但不适合做对线程有具体要求的工作。例如：初始化UI界面，必须发生在主线程。**

- unsubscribe(): Subscriber 同时实现了另一个接口 Subscription ，Subscription 接口只提供了两个方法：
  ```
  public interface Subscription {
          void unsubscribe();
          boolean isUnsubscribed();
  }
  ```
Subscriber 对这两个方法分别做了实现，isUnsubscribed() 用于判断Subscriber 是否已经订阅 ，而 unsubscribe() 方法用于 取消订阅。在 subscribe() 之后， Observable 会持有 Subscriber 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如 onPause() onStop() 等方法中）调用 unsubscribe() 来解除引用关系，以避免内存泄露的发生。

#### 创建 Observable
Observable 提供了大量的方法来进行 数据变换，下面我们借用 [Rxjava学习:初识Rxjava](http://www.jianshu.com/p/264a1322669a) 一文中的例子来对其各个API进行理解：
```
Observable.from(folders)
    .flatMap(new Func1<File, Observable<File>>() {
        @Override
        public Observable<File> call(File file) {
            return Observable.from(file.listFiles());
        }
    })
    .filter(new Func1<File, Boolean>() {
        @Override
        public Boolean call(File file) {
            return file.getName().endsWith(".png");
        }
    })
    .map(new Func1<File, Bitmap>() {
        @Override
        public Bitmap call(File file) {
            return getBitmapFromFile(file);
        }
    })
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) {
            imageCollectorView.addImage(bitmap);
        }
    });
```
- flatMap() : 通过传入的对象，根据需求 ，创建新的Observable 对象，并代替现有的Observable对象完成剩下的所要执行的方法。

- map() : 将传入的对象转换成另一个对象,是可以被直接发送到了Subscriber 的回调方法中使用的。如同上面示例中那样。

- filter() : 过滤筛选。 通过事件方法进行筛选，将符合要求的数据传递到下一个事件。例如示例中：符合**file.getName().endsWith(".png”)** 要求的数据才会传递。

#### 调用 subscribe() 进行订阅
创建了 Observable 和 Observer 之后，再用 subscribe() 方法将它们联结起来，整条链子就可以工作了。代码形式很简单：
```
observable.subscribe(observer);
```
同时subscribe()方法也提供了其他的重载以实现不完整定义的回调。
```
// 自动创建 Subscriber ，并使用 onNext 来定义 onNext()
subscribe(final Action1<? super T> onNext) 
// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError) 
// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
subscribe(final Action1<? super T> onNext, final Action1<Throwable> onError, final Action0 onCompleted)
```

### 总结：

 讲到现在，对Rxjava的使用也有了清晰的了解，在这里并没有提到Scheduler的使用，我觉得Scheduler  对于Rxjava是十分重要的一部分，我打算在下一篇中做详细的介绍，并对Observable其他重要的API方法进行介绍。