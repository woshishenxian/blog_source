---
title: Rxjava学习：初识Rxjava
date: 2016-08-23 10:32:39
tags: [Rxjava, RxAndroid] 
description: 
categories: Rxjava学习
---


### 前言
在过去的一年里，出现大量优秀的框架，火的不行，我也是从网上了解到Rx，特意搜索了相关文章，我也是通过它初步了解了Rxjava, 通过使用也慢慢领会到了其独特的魅力。
<!-- more-->
> *先谈谈我对Rxjava的认识*

首先说一下使用的感受：作为一只程序猿，最烦的就是那无穷无尽的for循环，以及那看起来都有点丧心病狂的 格式缩进，而如果使用Rxjava来进行数据处理的话，完全没有这方面的问题，你只要依次实现每一个方法就可以迅速获得同样的效果。


这里借鉴一下 [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)一文中的例子，来给大家说明一下： 

*假设有这样一个需求：界面上有一个自定义的视图 imageCollectorView ，它的作用是显示多张图片，并能使用 addImage(Bitmap) 方法来任意增加显示的图片。现在需要程序将一个给出的目录数组 File[] folders 中每个目录下的 png 图片都加载出来并显示在 imageCollectorView 中。需要注意的是，由于读取图片的这一过程较为耗时，需要放在后台执行，而图片的显示则必须在 UI 线程执行。常用的实现方式有多种，我这里贴出其中一种：*

```
new Thread() {
    @Override
    public void run() {
        super.run();
        for (File folder : folders) {
            File[] files = folder.listFiles();
            for (File file : files) {
                if (file.getName().endsWith(".png")) {
                    final Bitmap bitmap = getBitmapFromFile(file);
                    getActivity().runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            imageCollectorView.addImage(bitmap);
                        }
                    });
                }
            }
        }
    }
}.start();
```
而如果使用 RxJava ，实现方式是这样的：
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

这样，大家就可以理解， RxJava 的优秀在于其可读性，只要熟悉编写习惯后不管过多久再来看这部分代码，仍然可以很快的理解，而上面的各种for循环写出来的代码，并不具备这样的属性。

>*下面就来简单的讲一下Rxjava的使用*

### 引入RxJava

RxJava 支持Java 6或者更新的版本,引入前请更新到Java 6或以上版本。

```
compile 'io.reactivex:rxandroid:1.2.1'
compile 'io.reactivex:rxjava:1.1.8'
```
RxAndroid 是 Android 开发中使用 RxJava 必备元素，虽然里面只是提供了简单的两个功能。 AndroidSchedulers.mainThread() 
和 AndroidSchedulers.handlerThread(handler) ，但这确是 Android 开发中最核心的功能之一。

### Rxjava基本概念

下面来谈谈Rxjava几个基本概念的介绍:
- Observer（观察者）
- Observable （被观察者）
- subscribe(* observer *) （订阅）

>Observer（观察者）
Observer 提供一组不同的事件，并对各个事件进行实现，不关心事件何时发生，只决定事件触发的时候将有怎样的行为。

本质是一个接口，提供了三个方法，需要实现。

- onNext(Object o): 事件队列中，RxJava 每完成一次事件都会通过调用onNext(Object o), 对结果进行相应的操作。

- onCompleted(): 当整个事件队列完成所有的数据处理时，其将被调用。

- onError(): 在整个事件队列中，只要有一个事件处理出现问题，其将被调用，并放弃对剩下事件的处理。

```
Subscriber<String> subscriber = new Subscriber<String>() {
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

> Observable（被观察者）
Observable 提供事件触发的场景，即决定什么时候触发事件和此时该触发什么事件，不关心事件的具体实现。

其提供了多种初始化，创建对象的方法：
- create()
```
//使用Observable.create()创建被观察者
Observable observable1 = Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("Hello");
                subscriber.onNext("Wrold");
                subscriber.onCompleted();
            }
        });
```

- just(T...)
```
Observable observable2 = Observable.just("Hello", "World");
```
- from(T[]) 
```
String [] words = {"Hello", "World"};
Observable observable3 = Observable.from(words);
```
或者
```
List<String> list = new ArrayList<String>();
list.add("Hellow");
list.add("Wrold");
Observable observable4 = Observable.from(list);
```

>subscribe(* observer *) （订阅）
创建Observable和Observer之后，通过subscribe()方法将它们结合起来。

### 小结

对Rxjava作了一个简单的介绍，从基础知识开始讲起可能比较枯燥，没有多少实践，但是基础的重要性不言而喻，尤其对于Rxjava来说，其代码的编写习惯与我们平常不太一样，其链式的风格更加简洁，当你慢慢习惯，你将领会其独特的魅力。
