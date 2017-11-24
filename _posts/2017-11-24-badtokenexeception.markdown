---
layout:     post
title:      "[转]一个诡异的BadTokenException"
date:       2017-11-24 12:00:00
author:     "Desmond"
catalog: true
tags:
    - Android
    - Exception
---



原文链接https://blog.desmondyao.com



## 一个诡异的BadTokenException



在我们的Bugly上一直有一个排名较高的崩溃`android.view.WindowManager$BadTokenException`，堆栈是这样的：

```
android.view.WindowManager$BadTokenException Unable to add window -- token android.os.BinderProxy@432011d8 is not valid; is your activity running?
  android.view.ViewRootImpl.setView(ViewRootImpl.java:594)
  android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:269)
  android.view.WindowManagerImpl.addView(WindowManagerImpl.java:69)
  android.app.ActivityThread.handleResumeActivity(ActivityThread.java:3099)
  android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2350)
  android.app.ActivityThread.access$1100(ActivityThread.java:139)
  android.app.ActivityThread$H.handleMessage(ActivityThread.java:1256)
  android.os.Handler.dispatchMessage(Handler.java:102)
  android.os.Looper.loop(Looper.java:136)
  android.app.ActivityThread.main(ActivityThread.java:5315)
  java.lang.reflect.Method.invokeNative(Native Method)
  java.lang.reflect.Method.invoke(Method.java:515)
  com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:864)
  com.android.internal.os.ZygoteInit.main(ZygoteInit.java:680)
  dalvik.system.NativeStart.main(Native Method)
```

这个是一个非常正常的Activity启动堆栈，一脸黑线。StackOverflow上一搜这个崩溃基本都是Dialog有关，那么这个是一个什么诡异的情况，我们研究了好几次，都没有什么结果。最近偶然发现了一点蛛丝马迹，顺藤摸瓜下去，终于找到了原因。

首先从源码来探究一下这个BadTokenException是怎么出来的吧：

## 里面的”Token”是什么？

我们这个崩溃出现的地方全部都是在启动的第一个Activity中（即Launcher点击图标启动Activity），那就从这部分代码开始研究。

每一个`ActivityRecord`里面都有一个`appToken`变量，它是一个`Binder`对象，主要用于维持该Activity在AMS里与WindowManager之间的联系。它在`ActivityRecord`的构造函数中被初始化，通过调用`WindowManager.addAppToken`方法将该Token注册到WindowManagerService里面。

那么`ActivityRecord`是什么时候被初始化的呢？每一个Activity都在AMS里面有一个对应的ActivityRecord，在`startActivity`中的第一阶段就被初始化，并向WindowManagerService添加Token，我画了一张图来表示这个过程：

[![1](https://blog.desmondyao.com/image/app-bad-token/1.png)](https://blog.desmondyao.com/image/app-bad-token/1.png)

在`WindowManagerService.addAppToken`函数中，它对传入的token进行了一层包装，为了理解后面的内容，这里简单介绍一下这个函数：

(代码做了大量精简，有兴趣的同学可以自己阅读）

```
//WindowManagerService
public void addAppToken(int addPos, IApplicationToken token, int taskId, int stackId,
        int requestedOrientation, boolean fullscreen, boolean showForAllUsers, int userId,
        int configChanges, boolean voiceInteraction, boolean launchTaskBehind) {
        // 一些七七八八的代码

        AppWindowToken atoken = findAppWindowToken(token.asBinder());
        if (atoken != null) {
            Slog.w(TAG, "Attempted to add existing app token: " + token);
            return;
        }
        atoken = new AppWindowToken(this, token, voiceInteraction);
        // 对Token的初始化和赋值操作
        task.addAppToken(addPos, atoken);

        mTokenMap.put(token.asBinder(), atoken); //存入mTokenMap里

        //一些七七八八的代码
}
```

它根据传入的token新建了一个`AppWindowToken`对象，然后以传入的token为key，新建的对象为值放入了一个Map中。

我们再配合源码，可以看到先添加要启动的Activity的Token到WMS中，之后才会做下面几个操作：

- 处理已Resumed的、正在Paused中的Activity，让他们进入Paused状态；
- 以`ActivityThread.main()`为入口启动进程，初始化Context和Application；
- 启动目标Activity。

> 可以参考部分[罗升阳的Activity启动流程](http://blog.csdn.net/luoshengyang/article/details/6689748)，他分析的很好，可惜里面的Android版本有点老了。有兴趣的同学可以再看新的代码，我的分析是根据6.0的源码的。

现在回头看看报错是在哪一步：handleResumeActivity。那必然是在上面添加Token之后的事情了，我们追一下看看发生了什么：

堆栈最后一步 ViewRootImpl.addView

```
//ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {

        //七七八八的代码

        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
            getHostVisibility(), mDisplay.getDisplayId(),
            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
            mAttachInfo.mOutsets, mInputChannel);

        //七七八八的代码

        if (res < WindowManagerGlobal.ADD_OKAY) {
            mAttachInfo.mRootView = null;
            mAdded = false;
            mFallbackEventHandler.setView(null);
            unscheduleTraversals();
            setAccessibilityFocus(null, null);

            switch (res) {
                case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                    // 就是这里崩了
                    throw new WindowManager.BadTokenException(  
                            "Unable to add window -- token " + attrs.token
                            + " is not valid; is your activity running?");
```

这里的`mWindowSession`，它是一个Binder，具体的实现位于`Session.java`中。`ViewRootImpl`在初始化的时候会调用`WindowManagerGlobal.getWindowSession()`对它进行初始化，并保存在`WindowManagerGlobal`的静态变量`sWindowSession`中。之后Activity就能够通过这个Session通知WindowManager添加、删除View。

在`Session.addToDisplay`函数中我们可以看到它是直接透传给了`WindowManagerService`，并调用它的`addWindow`函数，我们看看它里面是怎么返回的。

1. `WindowManagerService.addWindow``// WindowManagerService    public int addWindow(Session session, IWindow client, int seq,            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,            InputChannel outInputChannel) {            WindowToken token = mTokenMap.get(attrs.token); //从mTokenMap中拿传入token的对应AppWindowToken            if (token == null) { //妈个鸡，没找到                if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {                     Slog.w(TAG, "Attempted to add application window with unknown token "                          + attrs.token + ".  Aborting.");                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN; //这里就返回了                }                // 剩下的代码都不重要了            }`

我们看到，这里就返回了`WindowManagerGlobal.ADD_BAD_APP_TOKEN`，然后`ViewRootImpl`那里就跑到了switch分支里面，报出崩溃。

好的，这是之前所有的研究成果，根本毫无头绪，怎么token突然没了？！这个bug就暂时搁置了。

# 突然发现关键信息

说到这里，不得不感谢一个[StackOverflow](http://stackoverflow.com/questions/5854290/why-does-resuming-an-activity-in-android-cause-badtokenexception)，它里面一个回答给了我提示：

> The change in my app that makes this crash possible is android:noHistory=”true” in Activity declaration of AndroidManifest.xml.

没错，我们的启动Activity确实标记了`noHistory`。虽然后面经过研究，他猜测的原因错了，但是他提供了复现方法：在Application里面做一个特别耗时的行为，然后此时切回Launcher，等待一会，就会崩溃。

其实这个之前不是没有考虑过，我曾经试过在Application里面sleep 10秒，没有崩。然后这次我直接sleep 30秒，崩了。

那去除noHistory呢？没崩。

[![mengbi](https://blog.desmondyao.com/image/mengbi.jpeg)](https://blog.desmondyao.com/image/mengbi.jpeg)

官方文档说加了这个标志之后，不会被保留在Task中。这个标志是为什么加的其实我也不知道，一开始就有了。好的吧，那到底为什么加了它就崩了？这得研究一下吧。

其实光从这里，我也看不出什么，但是我在复现的时候，注意到了一个地方，sleep久的时候，也就是造成崩溃的时候，系统打出来的日志里面总会多出一行

> 09-11 17:08:51.917 873-963/? W/ActivityManager: Activity destroy timeout for ActivityRecord{426e9e20 u0 com.desmond.testapplication/.MainActivity t21}

每次只要有它，就一定会崩。那就有一个这个猜想：

> 因为设置了noHistory为true的Activity在Stop时(按Home键触发）会强制触发destroy操作，但是此时Activity的主线程卡在了Application中无法顺利进行下面的操作，**AMS判定Destroy超时，于是将它强制销毁了！**

## 追溯源码

首先我们看一下AMS对于设置了noHistory=true的Activity是怎么处理的，有一处关键的代码：

```
// ActivityStack.java
final void stopActivityLocked(ActivityRecord r) {

    if ((r.intent.getFlags()&Intent.FLAG_ACTIVITY_NO_HISTORY) != 0
            || (r.info.flags&ActivityInfo.FLAG_NO_HISTORY) != 0) { 
        if (!r.finishing) {
            if (!mService.isSleeping()) { // 设备没有进入Sleep
                if (requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null, //这里finish
                        "stop-no-history", false)) {
                    // Activity was finished, no need to continue trying to schedule stop.
                    adjustFocusedActivityLocked(r, "stopActivityFinished");
                    r.resumeKeyDispatchingLocked();
                    return;
                }
            } else {
                if (DEBUG_STATES) Slog.d(TAG_STATES, "Not finishing noHistory " + r
                        + " on stop because we're just sleeping");
            }
        }
    }
```

通过上面这段代码能够发现，对于noHistory的Activity，它确实会在stop的时候被AMS触发一个finish操作。那么这就能解释的通了，不过整个过程还是比较复杂，我将它缕一下。

先看看Destroy Timeout 是哪里发出来的：

```
//ActivityStack.java
final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
        //...
        if (r.finishing && !skipDestroy) {
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to DESTROYING: " + r
                    + " (destroy requested)");
            r.state = ActivityState.DESTROYING;
            Message msg = mHandler.obtainMessage(DESTROY_TIMEOUT_MSG, r);
            mHandler.sendMessageDelayed(msg, DESTROY_TIMEOUT);
        }
        //...
}
```

注意这个`mHandler`是在AMS进程里的Handler，它使用的是`ActivityManagerService`里面成员变量`mHandler`的`Looper`，不会被Client-Side（即APP进程的操作所影响）。它在调用destroyActivityLocked的时候会先给Handler发一个延迟的TIMEOUT_MSG，如果client端顺利执行完了destroy，则会通知ActivityStack移除这个MSG，否则它就会被执行。

ActivityStack在收到这个消息以后会做一系列操作，可以参考这个图，我们可以看到最后它会把WindowManagerService中存的appToken给remove掉！

[![2](https://blog.desmondyao.com/image/app-bad-token/2.png)](https://blog.desmondyao.com/image/app-bad-token/2.png)

然后注意，这个时候我们的Application还卡着呢！它现在卡在`ActivityThread.handleBindApplication`里面，换句话说，它的Handler卡在处理`BIND_APPLICATION`中了，无法处理之后的消息。**自然所有的LAUNCH/RESUME/DESTROY全部都处理不了!** 那么这个时候WindowToken会被Timeout移除，接下来它在处理排队中的消息时，处理到RESUME，走到最开始说的ActivityThread.handleResumeActivity=>ViewRootImpl.addView，发现竟然找不到我的Token了！崩了。

没错，配合log，配合现象，这就是这个bug的唯一解释。

解释了后半部分，前半部分还不太清楚。DESTROY_TIMEOUT_MSG发送出来的呢？换句话说，`stopActivityLocked`是谁调用的呢？既然研究了就继续看一下。

按下Home键实际上是启动LauncherActivity，它是一个启动另外一个进程Activity的过程。我们可以通过按下Home键打出的`ActivityManager`的Log看出来：

> ActivityManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.HOME]

在LauncherActivity走到Resume过程的时候，`ActivityThread.handleResumeActivity`里面会调用`Looper.myQueue().addIdleHandler(new Idler());`添加一个`IdleHandler`，当Looper消息循环结束，进入空闲状态时会触发它的回调，象征着我现在客户端的主进程已经”Idle”了，随时待命。在这个IdleHandler里面调用`ActivityManagerService.activityIdle`函数。这里触发后续的操作，如图所示：

[![3](https://blog.desmondyao.com/image/app-bad-token/3.png)](https://blog.desmondyao.com/image/app-bad-token/3.png)

ps：这也印证了A start B，肯定是B的resume走完，再走A的stop的。

## 另外的尝试

之前分析到，是在`handleResumeActivity`的时候才向WindowManagerService添加View的，那么如果是其他声明了noHistory=true的Activity在`onCreate`中做超长耗时，并按了Home键出去，会不会一样崩呢？

各位看官可以自己写一个Demo来验证一下。

## 解决办法

要说根本原因，肯定是不能再主线程里面做太多事情，特别是会在Application与Activity的onCreate阶段。如果说要把这个Bug解了，就把noHistory去掉，那stop不会强制触发finish，自然就没有DESTROY_TIMEOUT，AMS就不会强制干掉你的WindowToken了。