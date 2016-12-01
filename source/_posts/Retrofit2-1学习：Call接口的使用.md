---
title: Retrofit2.1学习：Call接口的使用
date: 2016-11-14 15:33:37
tags:
categories: Retrofit2.1
---

#### 前言：
  前面提到 [Retrofit与Rxjava 的结合使用]() ,今天我们就来说一说Retrofit原生自带的Call接口的使用,本文主要针对Retrofit 2.0 的开发使用，对于2.0之前的版本的使用不做讨论。
<!-- more-->
#### 添加INTERNET权限
```
<uses-permission android:name="android.permission.INTERNET" />
```
在Retrofit 2.0中，当你调用call.enqueue或者call.execute，未添加权限，将立即抛出SecurityException，如果你不使用try-catch会导致崩溃。

#### Call 接口的实例化

```
public interface GitHubService {
  @GET("list")
  Call<List<Repo>> listRepos();
}
```
创建service 的方法和OkHttp的模式一模一样。

```
GitHubService service = retrofit.create(GitHubService.class);
```

如果要调用同步请求，只需调用execute；而发起一个异步请求则是调用enqueue。

#### 同步请求
```
Call<List<Repo>> call = service. listRepos();
List<Repo> repo = call.execute();
```

以上的代码会阻塞线程，因此你不能在安卓的主线程中调用，不然会面临NetworkOnMainThreadException。如果你想调用execute方法，请在开启子线程执行。

#### 异步请求
```
Call<Repo> call = service.listRepos("user");
call.enqueue(new Callback<Repo>() {
    @Override
    public void onResponse(Call<List<Repo>> call,Response<List<Repo>> response) {
        // Get result Repo from response.body()
    }
 
    @Override
    public void onFailure(Call<List<Repo>> call,Throwable t) {
 
    }
});
```
以上代码发起了一个在后台线程的请求并从response 的response.body()方法中获取一个结果对象。注意这里的onResponse和onFailure方法是在主线程中调用的。

这里建议你使用enqueue，它最符合 Android OS的习惯。

#### 取消正在进行中的业务

service 的模式变成Call的形式的原因是为了让正在进行的事务可以被取消。要做到这点，你只需调用call.cancel()。
```
call.cancel();
```

#### Callback 回调的使用

onResponse的调用，不仅仅是成功返回结果时调用，存在问题时也会被调用，这里查看了很多关于出现这个问题的解决方法，然而并没有给出一个明确的方案，之后查看了一下Retrofit2.0 提供的官方API解释，如下：

> An HTTP response may still indicate an application-level failure such as a 404 or 500. Call Response.isSuccessful() to determine if the response indicates success.
HTTP response 仍然可以指示应用程序级故障，如404或500。调用Response.isSuccessful()，以确定是否该响应指示成功。

```
@Override
    public void onResponse(Call<List<Repo>> call,Response<List<Repo>> response) {
        // Get result Repo from response.body()
       if(response.isSuccessful()){
           List<Repo> repos = response.body();
          //对数据的处理操作
       }else{
        //请求出现错误例如：404 或者 500
       }
    }
```

而关于 onFailure方法的调用，网上也并没有给出明确的调用时机，而官方文档给出解释：
>Invoked when a network exception occurred talking to the server or when an unexpected exception occurred creating the request or processing the response.
当连接服务器时出现网络异常 或者 在创建请求、处理响应结果 的时候突发异常 都会被调用。

通过自己测试发现了几种调用情况：**GSON解析数据转换错误**，**手机断网或者网络异常**。

```
    @Override
    public void onFailure(Call<List<Repo>> call,Throwable t) {
      
    }
```