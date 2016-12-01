---
title: Android性能优化方法：内存泄露优化
date: 2016-10-28 15:29:15
tags:
categories: android开发艺术谈说
---

### 前言
内存泄露在开发工程中是一个需要重视的问题，但是由于内存泄露问题对开发人员的经验和开发意识有较高的要求，因此这也是开发人员最容易犯的错误之一。内存泄露的优化分为两个方面，一方面是在开发过程中避免写出有内存泄露的代码，另一方面是用过一些分析工具比如MAT来找出潜在的内存泄露的代码继而解决。本届主要介绍一些常见的内存泄露的例子，通过这些例子读者可以很好的理解内存泄露的发生场景并积累规避内存泄露的经验。
<!-- more-->
### 场景一：静态变量导致内存泄露

下面这种情形是一种最简单的内存泄露，相信读者都不会这么干，下面代码将导致Activity无法正常销毁，一次静态变量sContext引用了它。
```
public class MainActivity extends Activity {
	private static final String TAG = "MainActivity";

	private static Context sContext;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		sContext = this;
	}
}
```
上面代码也可以改造一下，如下所示。sView是一个静态变量，它内部持有了当前Activity,所以Activity仍然无法释放。
```
public class MainActivity extends Activity {
	private static final String TAG = "MainActivity";

	private static View sView;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		sView = new View(this);
	}
}
```
### 场景2：单利模式导致的内存泄露
静态变量导致的内存泄露都太过于明显，相信读者都不会犯这种错误，而单例模式所带来的内存泄露使我们容易忽视的，如下所示。首先提供一个单例模式的TestManager,TestManager 可以接受外部的注册并将外部的监听器储存起来。
```
public class TestManager {
	private List<OnDataArrivedListener> mOnDataArrivedListeners = new ArrayList<OnDataArrivedListener>();

	private static class SingletonHolder {
		public static final TestManager INSTANCE = new TestManager();
	}

	private TestManager() {
	}

	public static TestManager getInstance() {
		return SingletonHolder.INSTANCE;
	}

	public synchronized void registerListener(OnDataArrivedListener listener) {
		if (!mOnDataArrivedListeners.contains(listener)) {
			mOnDataArrivedListeners.add(listener);
		}
	}

	public synchronized void unregisterListener(OnDataArrivedListener listener) {
		if (mOnDataArrivedListeners.contains(listener)) {
			mOnDataArrivedListeners.remove(listener);
		}
	}

	interface OnDataArrivedListener {
		public void onDataArrived(Object data);
	}
}
```
接着再让Activity视线OnDataArrivedListener 接口并向TestManager注册监听，如下所示。下面的代码由于缺少解注册的曹锁所以会引起内存泄露，泄露的原因是Activity的对象被单例模式的TestManager所持有，而单例模式的特点是声明周期和Application保持一致，因此Activity对象无法被及时释放。
```
public class MainActivity extends Activity implements OnDataArrivedListener{
	private static final String TAG = "MainActivity";


	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		TestManager.getInstance().registerListener(this);
	}

	@Override
	public void onDataArrived(Object data) {
		// TODO Auto-generated method stub
		
	}
}
```
### 场景3：属性动画导致内存泄露
从Android3.0开始，Google提供了属性动画，属性动画中有一类无线循环的动画，如果Activity中播放此类动画且没有在onDestory中去停止动画，那么动画会一直播放下去，尽管已经无法再界面上看到动画效果了，冰鞋这个时候Activity的View会被动画持有，而View又持有了Activity，导致最终Activity无法被释放。下面的动画就是无限动画，会泄露当前Activity,解决方法在Activity的onDestory中调用animator.cancel来停止动画。
```
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		TestManager.getInstance().registerListener(this);
		Button button = (Button) findViewById(R.id.button);
		ObjectAnimator animator = ObjectAnimator.ofFloat(button, "rotation", 0,
				360).setDuration(2000);
		animator.setRepeatCount(ValueAnimator.INFINITE);
		animator.start();
	}
```

### 一些性能优化建议
- 避免创建过多的对象。
- 不要过多使用枚举，枚举占用的内存空间要比整数大。
- 常量请使用 static final 来修饰。
- 使用一些Android特有的数据结构，比如SparseArray和Pair等，他们都具有更好地性能。
- 适当使用软引用和弱引用。
- 采用内存缓存和磁盘缓存。
- 尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄露。
