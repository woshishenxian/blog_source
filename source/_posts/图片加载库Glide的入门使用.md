---
title: 图片加载库Glide的入门使用
date: 2016-11-25 16:15:44
tags:
categories: Glide的使用
---
### 前言
在android开发如此盛行的今天，图片加载一直是开发的一个要点，市面上的图片加载库也是层出不穷，选择一个适合当前功能使用的图片加载库十分重要。
<!-- more-->
### 使用过的图片加载库
Universal Image Loader：最初开始使用的，足够强大，包含各种各样的配置，能满足你各种需求。
Picasso: Square出品，能和OkHttp搭配使用，唯一不足的是不能加载Gif图片。
Fresco：Facebook出的，十分强大，然而功能过于繁杂，使用起来比较麻烦。
Glide：专注于平滑图片加载，用于滚动时加载图片时相当流畅。

### Glide的使用

> 加载图片:load()-Glide 做了不同的重载用来支持本地及网络图片的加载

- load(String string)	string可以为一个文件路径、uri或者url
- load(Uri uri)                 Uri类型
- load(File file)  	        文件
- load(Integer resourceId)	资源Id,R.drawable.xxx或者R.mipmap.xxx
- load(byte[] model)	byte[]类型
- load(T model)	自定义类型

```
Glide.with(context).load(imageUrl).into(imageView);
```

Glide的with()方法不光支持Context，还支持Activity 和 Fragment，有一点很重要需要记住，就是传入的context类型影响到Glide加载图片的优化程度，Glide可以监视activity的生命周期，在activity销毁的时候自动取消等待中的请求。但是如果你使用Application context，你就失去了这种优化效果。

> 加载网络图片时,placeholder(Drawable drawable) 或者 placeholder(int resourceId)设置等待时显示的图片 ；通过 error(Drawable drawable) 或者 error(int resourceId)设置图片加载失败时显示的图片。

```
Glide.with(mContext).load(url).placeholder(drawable).error(drawable).into(imgView);
```
> Glide 提供了两种缩放方式 ：fitCenter()和 centerCorp()

```
Glide.with(mContext).load(url).fitCenter().into(imgView);
Glide.with(mContext).load(url).centerCrop().into(imgView);
```
缩放方式不多做介绍，其与scaleType 中的 fitCenter/centerCorp效果是一样的。

> 加载Gif图片

```
Glide.with(context).load(imageUrl).asGif().placeholder(drawable).error(drawable).into(imageView);
```
asGif() 方法判断所要加载的图片是否为 gif图，如果是则正常加载，如果不是则调用error()。也可以无需调用asGif() ,直接加载gif。

> 图片缓存

Glide 提供了多种不同的缓存策略，例如：skipMemoryCache(boolean skip) 判断是否直接跳过内存缓存
```
Glide.with(context).load(imageUrl).skipMemoryCache(true).into(imageView);
```
同时，对于磁盘缓存也做了多种扩展：
- DiskCacheStrategy.NONE 不做磁盘缓存
- DiskCacheStrategy.SOURCE 只缓存图像原图
- DiskCacheStrategy.RESULT 只缓存加载后的图像，即处理后最终显示时的图像
- DiskCacheStrategy.ALL 缓存所有版本的图像（默认行为）

调用diskCacheStrategy(DiskCacheStrategy strategy)设置磁盘缓存
```
	Glide.with(context).load(imageUrl).diskCacheStrategy(strategy).into(imageView);
```

> 图片加载优先级设定

Glide 引入优先级概念，提供了多种优先级设定：
- Priority.LOW
- Priority.NORMAL （默认优先级）
- Priority.HIGH
- Priority.IMMEDIATE  

调用priority(Priority priority) 指定优先级
```
Glide.with(context).load(imageUrl).priority(priority).into(imageView);
```

> Bitmap 的获取

```
Glide.with(mContext)
    .load(url) 
    .placeholder(drawable)
    .into(new SimpleTarget<Bitmap>(width, height) {
        @Override 
        public void onResourceReady(Bitmap bitmap, GlideAnimation anim) {
            // setImageBitmap(bitmap)
        } 
    };
```

### Glide 扩展使用

> 定义圆形图片加载

```
//圆形转换
public class GlideCircleTransform extends BitmapTransformation {
    public GlideCircleTransform(Context context) {
        super(context);
    }

    @Override protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        return circleCrop(pool, toTransform);
    }

    private static Bitmap circleCrop(BitmapPool pool, Bitmap source) {
        if (source == null) return null;

        int size = Math.min(source.getWidth(), source.getHeight());
        int x = (source.getWidth() - size) / 2;
        int y = (source.getHeight() - size) / 2;

        // TODO this could be acquired from the pool too
        Bitmap squared = Bitmap.createBitmap(source, x, y, size, size);

        Bitmap result = pool.get(size, size, Bitmap.Config.ARGB_8888);
        if (result == null) {
            result = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888);
        }

        Canvas canvas = new Canvas(result);
        Paint paint = new Paint();
        paint.setShader(new BitmapShader(squared, BitmapShader.TileMode.CLAMP, BitmapShader.TileMode.CLAMP));
        paint.setAntiAlias(true);
        float r = size / 2f;
        canvas.drawCircle(r, r, r, paint);
        return result;
    }

    @Override public String getId() {
        return getClass().getName();
    }
}
```
调用transform(Transformation... transformations) 方法完成转换
```
Glide.with(this).load(imageUrl).transform(new GlideCircleTransform(context)).into(imageView);
```

> 圆角图片加载

```
public class GlideRoundTransform extends BitmapTransformation {

    private static  float radius = 0f;

    public GlideRoundTransform(Context context) {
        this(context,15);
    }

    public GlideRoundTransform(Context context, int dp) {
        super(context);
        GlideRoundTransform.radius = Resources.getSystem().getDisplayMetrics().density * dp;
    }

    @Override protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        return roundCrop(pool, toTransform);
    }

    private static Bitmap roundCrop(BitmapPool pool, Bitmap source) {
        if (source == null) return null;

        Bitmap result = pool.get(source.getWidth(), source.getHeight(), Bitmap.Config.ARGB_8888);
        if (result == null) {
            result = Bitmap.createBitmap(source.getWidth(), source.getHeight(), Bitmap.Config.ARGB_8888);
        }

        Canvas canvas = new Canvas(result);
        Paint paint = new Paint();
        paint.setShader(new BitmapShader(source, BitmapShader.TileMode.CLAMP, BitmapShader.TileMode.CLAMP));
        paint.setAntiAlias(true);
        RectF rectF = new RectF(0f, 0f, source.getWidth(), source.getHeight());
        canvas.drawRoundRect(rectF, radius, radius, paint);
        return result;
    }

    @Override public String getId() {
        return getClass().getName() + Math.round(radius);
    }
}

```
使用方式与圆形相同。

> 当列表快速滑动时，暂停请求：pauseRequests()

```
Glide.with(context). pauseRequests();
```

> 当列表停止滑动时，重新请求：resumeRequests()

```
Glide.with(context).resumeRequests();
```

> 快速取消所有图片请求：clear()

```
Glide.with(context). clear();
```

### 使用过程中遇到的问题

> 当Glide用于将图片加载到自定义的Imageview时，需要设置占位图,加载的图片无法显示。

网上有很多解决方案，这里说一个我发现的一个简单方法： **设置一个显示动画**，这样就可以了，原因这里我就不详细深究了。
```
Glide.with(context).load(imageUrl).placeholder(drawable)
			.animate(R.anim.abc_fade_in).into(imgView);
```