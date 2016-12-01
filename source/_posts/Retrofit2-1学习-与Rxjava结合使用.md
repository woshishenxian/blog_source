---
title: Retrofit2.1学习:与Rxjava结合使用
date: 2016-11-11 09:54:39
tags:
categories: Retrofit2.1
---

### 前言：
相对于Retrofit原始Call接口的使用，网上现在更多的是介绍与Rxjava相结合多使用，今天我们也来聊聊关于Rxjava的使用。
<!-- more-->
> 1.对Rxjava的引入

首先需要导入 Rxjava 与 RxAndroid,以及Retrofit提供的适配接口
```
compile 'io.reactivex:rxandroid:1.2.1'
compile 'io.reactivex:rxjava:1.1.8'

compile 'com.squareup.retrofit2:adapter-rxjava:2.1.0'
```

接下来Retrofit对于Rxjava的适配做了封装，只需要一行代码就能完成引入。
```
Retrofit retrofit = new Retrofit.Builder().addConverterFactory(GsonConverterFactory.create())
        .baseUrl(baseUrl)
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create()).build();
```

> 2.Rxjava的结合使用

接下来只需要对Retrofit需要转换的java接口进行修改。
```
public interface ApiService {
    @GET("list")
    Observable<Vo> getData();}
```

这样就完成了对Rxjava的整个引入，关于Rxjava的如何使用本文就不做重点介绍，如果大家想要了解 可以看看这篇文章 《[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)》.
