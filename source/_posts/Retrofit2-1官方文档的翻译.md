---
title: Retrofit2.1官方文档的翻译
date: 2016-09-02 16:52:13
tags:
categories: Retrofit2.1
---
### 前言：

对于Retrofit也是刚开始接触，Retrofit 的使用也并没有太多的心得，网上有很多关于Retrofit的介绍，但是在我看来先从官方文档开始学习不失是一个好的选择。
<!-- more-->
---
### Retrofit
官方文档地址：http://square.github.io/retrofit/
>一种针对android和java的类型安全的http客户端。

### Introduction
Retrofit 将http API 装换成 一个java interface。
```
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
 Retrofit 类 生成一个  GitHubService interface 的实例。
```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```
每一个 Call 都要生成一个 同步或异步的http请求到远程web服务器。
```
Call<List<Repo>> repos = service.listRepos("octocat");
```

使用注解来描述http请求（包括）：
- 支持 替换URL参数 以及 查询参数的添加
- 支持 将整个对象转换成 request body
- Multipart 请求体 和 文件上传

### API Declaration

>通过对 interface 的方法及其参数进行注解来表示将执行一次怎样的请求。


##### 1.REQUEST METHOD
每一个方法都必须有一个由 请求方式 及其相对应的URL 组成的http注解。提供了5种注解方式：GET, POST, PUT, DELETE, 和 HEAD 。注解中指定对应的URL。
```
@GET("users/list")
```
URL中也可以指定查询参数。
```
@GET("users/list?sort=desc")
```

##### 2.URL MANIPULATION
可以通过方法中的可替换参数动态修改请求的URL。可替换部分可以由大括号加上字母来表示（例如：{id}）。相应的参数则必须由 @Path 并使用相同的字母串来注释出。
```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);
```
查询参数也可以被添加。
```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);
```
对于复杂的查询参数 可以直接组合成一个 Map 来添加。
```
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
```
##### 3.REQUEST BODY
可以直接指定一个对象(例子中的User)作为http 的 请求体，用@Body 来注解。
```
@POST("users/new")
Call<User> createUser(@Body User user);
```
该对象(例子中的User)将通过指定的converter(转换器)转换成 Retrofit 实例。在不添加 converter(转换器)的情况下，只有 RequestBody 可以使用。

##### 4.FORM ENCODED AND MULTIPART
方法也可以通过声明去发送 form-encoded 和 multipart 数据。方法在添加 @FormUrlEncoded 后才可以发送 Form-encoded 数据。而参数的注解要用 @Field 来完成，以 key-value的形式，包含 命名 以及 所要上传的值。
```
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```
方法 在添加 @FormUrlEncoded 后可以发送 Multipart 请求。方法参数要用 @Part 注解来声明。
```
@Multipart
@PUT("user/photo")
Call<User> updateUser(@Part("photo") RequestBody photo, @Part("description") RequestBody description);
```
#####5.HEADER MANIPULATION
可以使用 @Headers  来为 方法设置静态的header。
```
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();
```
```
@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("users/{username}")
Call<User> getUser(@Path("username") String username);
```
**注意：header 之间不会互相覆盖。具有相同名称的所有 header 将会被包括到请求中。**

一个请求的header 可以使用 @Header  来动态的修改，相应的参数必须要用@Header来提供。如果value 为 null，header会被忽略；除此之外，将对参数做toString处理后，使用处理后的结果。
```
@GET("user")
Call<User> getUser(@Header("Authorization") String authorization)
```
**每一个请求都需要添加 header的时候可以使用 OkHttp 的 interceptor 来指定。**

##### 6.SYNCHRONOUS VS. ASYNCHRONOUS
Call 实例 是可以 同步或异步执行的。每个实例是只能执行一次的，但调用clone() 方法 可以重新创建一个新的可使用的实例。
在Android系统中，回调会被执行在主线程中。而在java虚拟机中，回调会在http请求发生的线程中被执行。


### Retrofit Configuration
Retrofit 是 将你的API 接口都转换成可调用对象的 class。默认情况下，Retrofit设置好的平台是相当不错的，但其也允许自定义定制。

##### CONVERTERS
默认情况下，Retrofit只会反序列化 HTTP 的bodies 为 OKHTTP 的 ResponseBody类型，并且@Body 注解 也只接受 ResponseBody类型。

Converters 可以被添加支持其他的类型。
- Gson: com.squareup.retrofit2:converter-goon
- Jackson: com.squareup.retrofit2:converter-jackson
- Moshi: com.squareup.retrofit2:converter-mosh
- Protobuf: com.squareup.retrofit2:converter-protobuf
- Wire: com.squareup.retrofit2:converter-wire
- Simple XML: com.squareup.retrofit2:converter-simplexml
- Scalars (primitives, boxed, and String): 
- com.squareup.retrofit2:converter-scalars

下面有一个例子：
使用GsonConverterFactory类生成 GitHubService 接口的实例化，并支持Gson 反序列化数据。
```
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(GsonConverterFactory.create())
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

##### CUSTOM CONVERTERS
话太长不翻译了，大体意思就是：如果你不想使用Retrofit提供的converter，可以自定义一个以实现想要的效果。当你在创建一个adapter（适配器）的时候， 创建一个class 并 extends Converter.Factory class ，给adapter传递一个该类的实例就行。


##### GRADLE
```
compile 'com.squareup.retrofit2:retrofit:2.1.0'
```

##### PROGUARD
如果项目中添加了混淆，就把下面的文字添加到混淆文件中举行：

```
# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on RoboVM on iOS. Will not be used at runtime.
-dontnote retrofit2.Platform$IOS$MainThreadExecutor
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions
```