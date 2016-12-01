---
title: 消息栏通知Notification的那些事儿
date: 2016-12-01 09:26:12
tags: Notification
---
#### 前言
最近在开发过程中，用到了Notification的功能，需要实现一个自定义的布局。实现方式相信大家也都知道，就是通过RemoteViews来实现。但是在开发过程中还是遇到不少问题。
<!-- more-->
首先，还是有必要说明一下，**RemoteViews**主要被用于AppWidget和Notification，它描述一个在其它进程中显示的View。

#### RemoteViews 使用过程中遇到的问题

> 首先是关于 RemoteViews 设置的问题

```
Notification notification = new NotificationCompat.Builder(this)
                .setContent(remoteViews)
                .setSmallIcon(R.mipmap.ic_launcher)
                .setTicker("您有新短消息，请注意查收！").build();
```
NotificationCompat.Builder是有提供setContent的方法来设置RemoteViews的。然而存在一个问题，通过setContent()方法获得的Notification是固定高度的。如果View的高度比默认高度要大的话，就会有一部分显示不出来。

那么如何设置能够不存在高度限定呢？ 实际上也简单。
```
if(android.os.Build.VERSION.SDK_INT >= 16) {
            notification.bigContentView = remoteViews;
        }
 notification.contentView = remoteViews;
```
android 在sdk16的时候引入了通知的展开和收起的功能，通过两个手指滑动操作，这里 bigContentView 表示展开时的显示的View，而contentView 表示收起时显示的View。我们可以通过设置 bigContentView 来完全显示，但仍然需要注意几个点：
- bigContentView是在sdk16的时候才引入的，所以一定不要忘记判断一下。对于小于16，的则只能定高了。

- 就算是bigContentView 也是有最大高度的，其最大高度是256dp，而contentView 默认情况下高度为64p。

- Notification 使用过程中， .setSmallIcon、setTicker 是一定要设置的。

> 自定义View 对背景色的适配

Android通知栏的背景各式各样，不同的ROM都适配有不同的背景。不同的Android版本通知栏背景也不一样，一旦我们为自定义通知上的元素设置了特定背景或颜色，就肯定会带来兼容性问题。

由此可以想到一种方法，将背景设为透明的，然后如果可以找到系统通知栏默认的文字样式，这样就可以显示出同样的效果了。


![屏幕快照 2016-11-13 13.01.56.png](/imgs/notification_03.png)

android 也确实提供了这样的样式，使用方式也简单,如下面这样设置即可。
```
android:textAppearance="@style/TextAppearance.StatusBar.EventContent.Title"
```
同时还提供了其他样式，如 TextAppearance.StatusBar.EventContent.Time，TextAppearance.StatusBar.EventContent.Info 等都可以试试。

然而这并没有结束，5.0以后的版本，android 又修改了新的默认的样式，所以就需要我们新建layout-v21，添加新的适配文件。


![屏幕快照 2016-11-13 13.27.23.png](/imgs/notification_04.png)

设置的方式并没有什么不同
```
android:textAppearance="@android:style/TextAppearance.Material.Notification.Title"
```
ok ，大功告成，来看一下效果。

白色背景：
![截屏_20161113_133535.png](/imgs/notification_01.png)

黑色背景：

![Screenshot_20161113-134026.png](/imgs/notification_02.png)

下面介绍一种大神的方案-大体的方式是： 通过获取系统通知栏Notification默认的标题颜色，然后比对色值，判断出通知栏背景的颜色是黑色域还是白色域，之后就可以选取相应的自定义通知栏。

对这个方案也测试了一下，vivo测试机 通知栏背景色是白色的，然而获得的标题的颜色也是白色的，也就是说 还是会出现 不可预知的问题。总之，学习一下还是可以的。

```
//逻辑相应代码
public class NoficationBar {

    private static final double COLOR_THRESHOLD = 180.0;

    private String DUMMY_TITLE = "DUMMY_TITLE";
    private int titleColor;

    public NoficationBar() {
    }

    //判断是否Notification背景是否为黑色
    public boolean isDarkNotificationBar(Context context) {
        return !isColorSimilar(Color.BLACK, getNotificationTitleColor(context));
    }



    //获取Notification 标题的颜色
    private int getNotificationTitleColor(Context context) {
        int color = 0;
        if (context instanceof AppCompatActivity) {
             color = getNotificationColorCompat(context);
        } else {
            color = getNotificationColorInternal(context);
        }
        return color;
    }

    //判断颜色是否相似
    public boolean isColorSimilar(int baseColor, int color) {
        int simpleBaseColor = baseColor | 0xff000000;
        int simpleColor = color | 0xff000000;
        int baseRed = Color.red(simpleBaseColor) - Color.red(simpleColor);
        int baseGreen = Color.green(simpleBaseColor) - Color.green(simpleColor);
        int baseBlue = Color.blue(simpleBaseColor) - Color.blue(simpleColor);

        double value = Math.sqrt(baseRed * baseRed + baseGreen * baseGreen + baseBlue * baseBlue);
        return value < COLOR_THRESHOLD;

    }

    //获取标题颜色
    private int getNotificationColorInternal(Context context) {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        builder.setContentTitle(DUMMY_TITLE);
        Notification notification = builder.build();

        ViewGroup notificationRoot = (ViewGroup) notification.contentView.apply(context, new FrameLayout(context));
        TextView title = (TextView) notificationRoot.findViewById(android.R.id.title);
        if (title == null) { //如果ROM厂商更改了默认的id
            iteratorView(notificationRoot, new Filter() {
                @Override
                public void filter(View view) {
                    if (view instanceof TextView) {
                        TextView textView = (TextView) view;
                        if (DUMMY_TITLE.equals(textView.getText().toString())) {

                            titleColor = textView.getCurrentTextColor();
                        }
                    }
                }
            });
            return titleColor;
        } else {

            return title.getCurrentTextColor();
        }
    }


    private int getNotificationColorCompat(Context context) {
        NotificationCompat.Builder builder = new NotificationCompat.Builder(context);
        Notification notification = builder.build();
        int layoutId = notification.contentView.getLayoutId();
        ViewGroup notificationRoot = (ViewGroup) LayoutInflater.from(context).inflate(layoutId, null);
        TextView title = (TextView) notificationRoot.findViewById(android.R.id.title);
        if (title == null) {
            final List<TextView> textViews = new ArrayList<>();
            iteratorView(notificationRoot, new Filter() {
                @Override
                public void filter(View view) {
                    if (view instanceof TextView) {
                        textViews.add((TextView) view);
                    }
                }
            });
            float minTextSize = Integer.MIN_VALUE;
            int index = 0;
            for (int i = 0, j = textViews.size(); i < j; i++) {
                float currentSize = textViews.get(i).getTextSize();
                if (currentSize > minTextSize) {
                    minTextSize = currentSize;
                    index = i;
                }
            }
            textViews.get(index).setText(DUMMY_TITLE);
            return textViews.get(index).getCurrentTextColor();
        } else {
            return title.getCurrentTextColor();
        }

    }


    private void iteratorView(View view, Filter filter) {
        if (view == null || filter == null) {
            return;
        }
        filter.filter(view);
        if (view instanceof ViewGroup) {
            ViewGroup container = (ViewGroup) view;
            for (int i = 0, j = container.getChildCount(); i < j; i++) {
                View child = container.getChildAt(i);
                iteratorView(child, filter);
            }
        }

    }

    interface Filter {
        void filter(View view);
    }

}
```

参考文章： http://www.jianshu.com/p/426d85f34561