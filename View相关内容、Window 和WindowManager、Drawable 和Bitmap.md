### 事件分发机制的一些结论

1.事件是一系列的事件，并不能单独出现

2.正常情况下事件只能被一给View拦截且消耗

3.某个View一旦决定拦截，那么这个事件序列都只能由它处理，并且它的onInterceptTouchEvent不会再被调用

4.某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件，那么同一事件序列的其他事件都不会交给它处理，并且事件将重新交给它的父元素去处理，即父元素的onTouchEvent会被调用

5.如果View不消耗除ACTION_DOWN以外的其他事件，那么这个点击事件会消失，此时父元素的onTouchEvent并不会被调用，并且当前的View可以持续收到后续事件，最终这些消失的点击事件会传递给Activity处理

6.ViewGroup默认不拦截任何事件

7.View没有onInterceptTouchEvent方法，一旦事件传递给它，它的onTouchEvent会被调用

8.View的OnTouchEvent默认返回true，除非是不可点击View，即clickable与longClickable同时为false。View的longClickable默认为false，clickable需要分情况，比如Button的clickable默认为true，TextView默认为false。

9.View的enable属性不影响onTouchEvent的默认返回值

### 滑动冲突的常规解决方式

##### 1.外部拦截法

事件先经过父容器拦截，如果父容器需要就拦截，不需要就不拦截，重写onInterceptTouchEvent方法。在重写方法是要注意ACTION_DOWN事件必须返回false，不拦截ACTION_DOWN事件，其次ACTION_MOVE事件，根据需求判断是否拦截，需要拦截返回true，不拦截返回false，最后是ACTION_UP事件，返回false以下为伪代码

```java
public boolean onInterceptTouchEvent(MotionEvent event){
    boolean intercepted = false;
    int x = event.getX();
    int y = event.getY();
    switch(event.getAction){
         case MotionEvent.ACTION_DOWN:
            intercepted = false;
            break;
         case MotionEvent.ACTION_MOVE:
            if(父容器需要){
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
         case MotionEvent.ACTION_UP:
            intercepted = false;
            break;   
    }
    mLastInterceptX = x;
    mLastInterceptY = y;
    return intercepted;
}
```

##### 2.内部拦截法

内部拦截是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交给父容器处理，这种拦截方法需要配合requestDisallowInterceptTouchEvent方法才能正常工作。首先需要重写view的dispatchTouchEvent方法

```java
//伪代码
public boolean dispatchTouchEvent(MotionEvent event){
    int x = event.getX();
    int y = event.getY();
    switch(event.getAction){
         case MotionEvent.ACTION_DOWN:
            parent.requestDisallowInterceptTouchEvent(true)
            break;
         case MotionEvent.ACTION_MOVE:
            int deltaX = x - mLastX;
            int deltaY = y - mLastY;
            if(父容器需要){
                parent.requestDisallowInterceptTouchEvent(false)
            } 
            break;
         case MotionEvent.ACTION_UP:
            break;   
    }
    mLastX = x;
    mLastY = y;
    return super.dispatchTouchEvent(event);
}
```

除了子元素的处理以外，父元素需要默认拦截除了ACTION_DOWN以外的其他事件，之所以父容器不能拦截ACTION_DOWN事件，是应为ACTION_DOWN事件在源码中不受FLAG_DISALLOW_INTERCEPT这个标记位控制，一旦父容器拦截了ACTION_DOWN事件，所有的事件无法传递给子元素，内部拦截就不能生效了。

```java
public boolean onInterceptTouchEvent(MotionEvent event){
    if(event.getAction == MotionEvent.ACTION_DOWN){
        return false;
    } else {
        return true;
    }
}
```

### View的工作机制

View的主要三大流程是measure, layout,draw,这三大流程需要掌握，另外View的构造方法，onAttach、onVisibility、onDetach等方法也需要熟练掌握

##### 1.View和Window是如何建立连接的

View和Window建立连接是通过ViewRoot和DecorView建立起链接。ViewRoot的实现类对应ViewRootImpl，它是链接WindowManager和DecorView的纽带，View的三大流程都是通过ViewRoot来完成的。在ActivityThread中，当Activity被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl和DecorView建立关系

View的绘制流程是从ViewRoot的performTraversals方法开始，经过measure，layout和draw三个过程最终将一个View绘制出来。performTraversals会依次调用performMeasure，performLayout，performDraw三个方法，performMeasure() ->measure()->onMeasure()一遍流程走完就完成了一次绘制，然后会重复这个过程，完成整个View树的遍历。performLayout->layout()->onLayout(),performDraw->draw()->dispatchDraw()->onDraw().

DecorView是个顶级View，是一个FrameLayout，一般情况下会包含一个竖直方向的LinearLayout，这个LinearLayout分为两部分，标题和内容，Activity的setContentView就是将View设置进内容部分。id叫content

```java
ViewGroup content = findViewById(android.R.id.content);
content.getChildAt(0);
```

事件都是先经过DecorView，然后才传递给我们的View

##### 2.MeasureSpec

MeasureSpec是一个32位的int值，高2位代表SpecMode，低30位代表SpecSize.

SpecMode分3类 UNSPECIFIED EXACTLY AT_MOST

##### 3.获取View的尺寸

1.Activity／View#onWindowFocusChanged 失去焦点和获得焦点都会调用，如果频繁的onPause和onResume的话，该方法会被频繁调用

```java
public void onWindowFocusChanged(boolean hasFocus){
    if(hasFocus) {
        int width = view.getMeasureWidth();
        int height = view.getMeasureHeight();
    }
}
```

2.view.post(runnable) 通过post可以将一个runnable投递到消息队列的末尾，然后等Looper调用此runnable的时候，view也已经初始化完毕了

```java
protected void onStart() {
        super.onStart();
        mView.post(new Runnable() {
            @Override
            public void run() {
                int width = mView.getMeasuredWidth();
                int Height = mView.getMeasuredHeight();
            }
        });
    }
```

3.ViewTreeObserver 当View树的状态发生改变或者View的可见性发生改变时会被调用，但也会被调用多次

```java
protected void onStart() {
        super.onStart();
        ViewTreeObserver observer = mView.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                mView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
                int width = mView.getMeasuredWidth();
                int Height = mView.getMeasuredHeight();
            }
        });
    }
```

##### 4.layout过程

Layout的作用是ViewGroup用来确定子元素的位置，当ViewGroup的位置被确定后，它在onLayout方法中会遍历所有的子元素并调用其layout方法，在layout 方法中onLayout方法又会被调用。layout 方法是确定View本身的位置，onLayout方法是确定所有子元素的位置

##### 5.draw过程

### 自定义View

##### 1.分类

##### 2.注意事项

1.尽量让view支持wrap_content

2.尽量支持padding

3.尽量不要使用Handler，view中有一系列的post可替代handler

4.在View中如果有线程或者动画需要及时停止，否者会造成内存泄漏

### Window和WindowManager

Window表示的是一个窗口的概念，是一个抽象类，实现类是PhoneWindow。创建Window需要使用WindowManager，WindowManager是访问Window的入口。Window 的具体实现在WindowManagerService中，WIndowManager和WindowManagerService的交互是一个IPC过程。Android的所有视图都是通过Window来呈现的，不管是Activity，Dialog，Toast，它们都是附加在Window上的，因此Window也是View的直接管理者。事件的分发都是由Window传递给DecorView，再由DecorView传递给View。

Window具象话为一个View，Window并不是实际存在的，是以一个View的形式存在。在WindowManager的定义里提供的三个接口方法addView，updateViewLayout以及romoveView都是针对View的。

Window大致分为三类，应用Window，子Window和系统Window。应用Window对应一个Activity，子Window不能单独存在，需要附属在特定的父Window中，比如常见的Dialog。系统Window是需要声明权限才能创建的Window，比如Toast和系统的状态栏。

Window是分层的，每个Window都有对应的z-ordered，层级大的会覆盖在层级小的window的上面。应用Window的层级范围在1-99，子Window的层级范围在1000-1999，系统Window在2000-2999。每个大的层级对应WindowManager.LayoutParams的type参数。

##### Activity与Dialog中Window的创建过程

1.创建Window

2.初始化DecorView并将视图添加到DevorView中

3.将DevorView添加到Window中并显示

##### Toast中的Window

Toast是基于Window实现的，由于具有定时取消的功能，内部采用了Handler。内部的IPC过程分为两类，第一类是Toast 访问NotificationManagerService，第二类是NotificationManagerService回调Toast的TN接口。Toast的显示隐藏都是通过NotificationManagerService实现的，NotificationManagerService运行于系统进程中，因此只能通过跨进程的方式显示隐藏Toast。TN是个Binder类，当NotificationManagerService需要控制Toast的时候，会跨进程回调TN中的方法，这个时候的TN运行在Binder线程池中，所以需要handler将其切换到当前线程中。由于使用Handler，所以Toast无法在没有Looper的线程中使用。对于非系统应用来说，最多存在50个Toast。

### Drawable和Bitmap

Drawable表示一种图像的概念，但是又不全是图片，通过颜色可以构造各式各样的图像效果。Drawable一般是通过xml定义，虽然可以使用代码，但是会稍显复杂。

BitmapDrawable 表示一张图片 对应bitmap标签

ShapeDrawable 对应shape标签

LayerDrawable 表示一种层次化的Drawable集合 对应layer-list

StateListDrawable 对应selector

LevelListDrawable 对应level-list Drawable集合，设置对应的等级显示不同的Drawable

TransitionDrawable 对应 transition 实现两个Drawable之间的淡入淡出效果

InsetDrawable 可以将其他的Drawable内嵌在自己当中，并可以在周围留出一定的距离

ScaleDrawable 对应 scale 根据自身等级缩放一定比例

ClipDrawable 对应 clip 根据自身的等级裁剪另一个Drawable

Drawable的核心方法就是draw 自定义Drawable其实就是draw。

Bitmap在Android中表示的是一张图片