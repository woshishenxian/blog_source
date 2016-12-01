---
title: Retrofit2.1学习:与okhttp3.0结合使用
date: 2016-10-31 09:15:36
tags:
categories: Retrofit2.1
---
![配图](/imgs/2149115-f2becc1a2a11fcfc.jpg)

### 1.添加日志输出


```
Retrofit retrofit = new Retrofit.Builder().client(new OkHttpClient.Builder().addNetworkInterceptor(
   new HttpLoggingInterceptor().setLevel(HttpLoggingInterceptor.Level.HEADERS)).build());
```
<!-- more-->
***
### 2.添加请求头信息

在 [Retrofit官方文档的翻译](http://www.jianshu.com/p/8cf33ad72211) 一文中有提到，**每一个请求都需要添加相同的 header的时候可以使用 OkHttp 的 interceptor 来指定。**

```
new Retrofit.Builder()
           .addConverterFactory(GsonConverterFactory.create())
           .client(new OkHttpClient.Builder()
                   .addInterceptor(new Interceptor() {
                       @Override
                       public Response intercept(Chain chain) throws IOException {
                           Request request = chain.request()
                                   .newBuilder()
                                   .addHeader("Accept", "application/vnd.github.v3.full+json")
                                   .addHeader("User-Agent", "Retrofit-Sample-App")
                                   .build();
                           return chain.proceed(request);
                       }
                   })

                   .build()
```
***
### 3.支持自签名的https 


首先这里要说一下，看到网上说了很多okhttp如何支持https的文章。首先要了解的是，okhttp默认情况下是支持https协议的网站的，比如 *https://www.baidu.com* 等，这些网站可以直接通过okhttp请求，不需要添加而外的支持。不过要注意的是，支持的https的网站基本都是CA机构颁发的证书，默认情况下是可以信任的。

然后我们今天要说的是自签名的https，什么叫自签名呢？就是自己通过keytool去生成一个证书，然后使用，并不是CA机构去颁发的。使用自签名证书的网站，大家在使用浏览器访问的时候，一般都是报风险警告，好在有个大名鼎鼎的网站就是这么干的，https://kyfw.12306.cn/otn/ ，点击进入12306的购票页面就能看到了。

在写本文之前也查看了很多资料，我呢，尽量把每一步都说的详细一点。

>SSL单向认证

加入证书源文件，证书可以放在assets 下面，或者raw 下面也是可以的：

![配图1](/imgs/2149115-a75ab976a221b88c.png)


- 构造keyStore对象， 然后将得到的CertificateStream放入到keyStore中。
```
InputStream certificateStream = mContext.getResources().openRawResource(R.raw.srca);

 KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());

 keyStore.load(certificateStream, "password".toCharArray());    
```
- 接下来利用keyStore去初始化我们的TrustManagerFactory.

```
TrustManagerFactory trustManagerFactory = 
          TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm()); 
trustManagerFactory.init(keyStore);
```
- 由trustManagerFactory.getTrustManagers获得TrustManager[]初 始化我们的SSLContext
```
//"TLS" SSL协议 由服务器端确定
SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init ( 
                null, 
                trustManagerFactory.getTrustManagers(), 
                null);
```
根据okhttp3.0以前的版本，上面这样写是没问题的，3.0以后版本API文档中，这样写道:
If necessary, you can create and configure the defaults yourself with the following code:
如果有必要，你可以参考下面的代码根据自己的需求创建和设置的默认值。

```
TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(
       TrustManagerFactory.getDefaultAlgorithm());
 trustManagerFactory.init((KeyStore) null);
 TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
     throw new IllegalStateException("Unexpected default trust managers:"
         + Arrays.toString(trustManagers));
   }
X509TrustManager trustManager = (X509TrustManager) trustManagers[0];

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, new TrustManager[] { trustManager }, null);
SSLSocketFactory sslSocketFactory =
        sslContext.getSocketFactory();

OkHttpClient client = new OkHttpClient.Builder()
       .sslSocketFactory(sslSocketFactory, trustManager);
       .build();
```
- 那根据上面的提供，我们来修改一下SSLContext的初始化：
```
TrustsManager[] trustManagers = trustManagerFactory.getTrustManagers();
X509TrustManager trustManager = chooseTrustManager(trustManagers);
if (trustManager != null) {   
     trustManager = new MyTrustManager(chooseTrustManager(trustManagers));
     } else {  
      trustManager = new UnSafeTrustManager();
     }
sslContext.init(null, new TrustManager[]{trustManager}, null);
```

```
private static X509TrustManager chooseTrustManager(TrustManager[] trustManagers) {
    for (TrustManager trustManager : trustManagers) {
        if (trustManager instanceof X509TrustManager) {
            return (X509TrustManager) trustManager;
        }    
  } 
  return null;
}
```

```
private static class MyTrustManager implements X509TrustManager {
    private X509TrustManager defaultTrustManager;
    private X509TrustManager localTrustManager;
    public MyTrustManager(X509TrustManager localTrustManager) throws NoSuchAlgorithmException, KeyStoreException {
        TrustManagerFactory var4 = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        var4.init((KeyStore) null);
        this.defaultTrustManager = chooseTrustManager(var4.getTrustManagers());
        this.localTrustManager = localTrustManager;
    }
    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) throws CertificateException {
    }
    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType) throws CertificateException {
        try {
            localTrustManager.checkServerTrusted(chain, authType);
        } catch (CertificateException ce) {
            defaultTrustManager.checkServerTrusted(chain, authType);
        }
    }
    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return new X509Certificate[0];
    }
}
```

- 最后，设置我们OkHttpClient即可。

```
OkHttpClient okHttpClient = new OkHttpClient.Builder()
        .sslSocketFactory(sslContext.getSocketFactory(), trustManager)).build();
```
到此为止，单向认证基本讲完了 ，接下来双向认证就相对简单了。

>SSL双向认证

核心代码：
```
//初始化keystore
KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
clientKeyStore.load(mContext.getAssets().open("zhy_client.bks"), "123456".toCharArray());

KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
keyManagerFactory.init(clientKeyStore, "123456".toCharArray());

sslContext.init(keyManagerFactory.getKeyManagers(), new TrustManager[]{trustManager}, null);
```
关于客户端证书文件，Java平台默认识别jks格式的证书文件，但是android平台只识别bks格式的证书文件。

***
### 4.认证机构认证后的https
>添加证书certificatePinner
certificatePinner(CertificatePinner certificatePinner) 的是在 由正式证书颁发机构认证的情况下，避免证书颁发机构的非法访问。

```
OkHttpClient client = new OkHttpClient.Builder()
    .certificatePinner(new CertificatePinner.Builder()
            .add("YOU API.com", "sha1/DmxUShsZuNiqPQsX2Oi9uv2sCnw=")
            .add("YOU API..com", "sha1/SXxoaOSEzPC6BgGmxAt/EAcsajw=")
            .add("YOU API..com", "sha1/blhOM3W9V/bVQhsWAcLYwPU6n24=")
            .add("YOU API..com", "sha1/T5x9IXmcrQ7YuQxXnxoCmeeQ84c=")
            .build())
```

>这是主机名验证
设置用于确认响应证书申请请求的主机名的HTTPS连接的验证。
如果不设置，默认的主机名校验将被使用。

```
okhttpBuilder.hostnameVerifier(HttpsFactroy.getHostnameVerifier(hosts));
```

最后，Retrofit与 okhttp3.0的结合：
```
retrofit = new Retrofit.Builder().client(client).build();
```

# 总结
关于okhttp的使用，本文不多说，主要是retrofit与okhttp相结合使用中用到的部分，本文也只是本人在 okhttp 与 retrofit 的结合使用中的一点总结，如果有不对的地方，望书友们指导。

***
参考文章：

- [Retrofit 2.0 超能实践，完美支持加密Https传输](http://www.jianshu.com/p/16994e49e2f6)
- [okHttp官方文档](http://square.github.io/okhttp/3.x/okhttp/)
- [Android Https相关完全解析 当OkHttp遇到Https](http://blog.csdn.net/lmj623565791/article/details/48129405)