---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(一)View体系          # 标题 
subtitle:   Android advance #副标题
date:       2018-07-03            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 坐标系
Android系统里面有两种坐标系：Android坐标系、View坐标系。
### 1.1 Android坐标系
![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/view/1.jpg?raw=true)
Android的坐标系是以手机上可见的屏幕左上角顶点为坐标系原点，但是xy轴的方向和我们以前知道的有所不同，需要注意，从原点向右为x轴正方向，而从原点向下为y轴正方向。

`android.view.MotionEvent`下面有两个方法`getRawX()`和`getRawY()`可以获得当前触摸位置的Android坐标系坐标。

### View坐标系
可以说Android坐标系是屏幕绝对坐标，View坐标系是viewgroup的相对坐标，这两者可以结合使用，做到更加精确的控制。
![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/view/2.png?raw=true)
| 方法 | 从属 | 含义 |
| ------ | ------ | ------ |
| getX() | MotionEvent | 触摸位置距离控件左边缘距离  |
| getY() | MotionEvent | 触摸位置距离控件上边缘距离 |
| getTop() | View | 当前view到其父布局上边缘距离 |
| getBottom() | View | 当前view下边缘到其父布局上边缘距离 |
| getLeft() | View | 当前view左边缘到其父布局左边缘距离 |
| getRight() | View | 当前view右边缘到其父布局左边缘边缘距离 |

## 2 源码解析Activity的view体系
### 2.1 view体系
我们都知道要在Activity的`onCreate()`方法里面用setContentView()方法加载布局，那布局在源码里是如何被加载出来的，来分析一下。
首先进入`setContentView`，看到方法如下：
```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
     *// @return Window The current window, or null if the activity is not visual.
     */       
public Window getWindow() {
    return mWindow;
}
```
这个方法里面首先先去调用getWindow()将布局塞入到获得的window对象里面，注释说明这个`mWindow`是当前的window，如果activity不可见就返回`null`。

再来看看window是如何被初始化出来的，我们在attach方法里面找到了赋值代码：
```java
mWindow = new PhoneWindow(this, window, activityConfigCallback);
```
mWindow就是PhoneWindow对象的实例，所以getWindow().setContentView()是调用了PhoneWindow对象的setContentView方法:
```java
// This is the view in which the window contents are placed. It is either
    // mDecor itself, or a child of mDecor where the contents go.
    ViewGroup mContentParent;
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);
            ...
        }
        ...
 }
```
该方法判断`mContentParent`是否存在，不存在就会调用 installDecor()方法去创建DecorView，存在的话就把布局加载到mContentParent中，按照注释，这个mContentParent就是DecorView本身或者DecorView的content部分。

再来看installDecor方法，会先判断DecorView是否存在，不存在先调用generateDecor创建DecorView，然后判断mContentParent是否存在，不存在就把调用generateLayout(mDecor)创建mContentParent。
而generateLayout(mDecor)又会加载一个R.layout.screen_title的布局，这里面有TitleView布局和ContentView布局。

![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/view/3.png?raw=true)

总结一下：Activity在首先会加载一个PhoneWindow对象，然后再去加载DecorView对象，DecorView会作为activity的根view存在，activity的setContentView最终会调用DecorView的setContentView方法，将布局加载到DecorView上，DecorView由title和content两部分组成。

### 2.2 DecorView添加到window
上文介绍了view的层次和DecorView的创建过程，现在需要将创建好的DecorView添加到window中去。

```java
 if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();
```
每当我们启动一个Activity的时候,都会执行`ActivityThread.main()`方法，main方法里面有一个Looper一直在在等待主线程的消息，可以看到sMainThreadHandler会接受一个handler,这是一个内部类H，继承自Handler。看一下它的handleMessage()方法，
```java
public void handleMessage(Message msg) {
     if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
            ....
            case RELAUNCH_ACTIVITY:
                    handleRelaunchActivityLocally((IBinder) msg.obj);
                    break;
            }
``` 
接收到启动activity的消息之后就会调用`handleRelaunchActivityLocally`,然后该方法会调用handleLaunchActivity(r, pendingActions, customIntent)，`handleLaunchActivity`又会调用performLaunchActivity(r, customIntent)，方法内部会调用mInstrumentation.callActivityOnCreate去回调生命周期方法`onCreate`，这样就完成DecorView的创建。

onCreate之后会走到onStart，`handleStartActivity`方法里面会走一些延迟加载的任务，保存状态等。

然后就是onResume,`handleResumeActivity`会执行将DecorView加载到window的操作，看一下源码：
```java
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest,boolean isForward,String reason) {
    ...

    if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }
            ...
    }
}
```
这个方法内会将onCreate时创建的DecorView通过`getWindowManager`返回的wm的实例方法` wm.addView(decor, l)`添加到WindowManager中，WindowManager的实例是WindowManagerImpl，它的addView又会调用WindowManagerGlobal的addView方法，通过ViewRootImpl实例的setView()方法，将DecorView塞到window里面。
```java
root = new ViewRootImpl(view.getContext(), display);

            view.setLayoutParams(wparams);

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);
```

## 参考资料
* 《Android进阶之光》
*  [Android中Activity启动过程探究](https://www.cnblogs.com/kross/p/4025075.html)
* [View 体系详解：View 的工作流程](https://shouheng88.github.io/2018/10/14/View%20%E4%BD%93%E7%B3%BB%E8%AF%A6%E8%A7%A3%EF%BC%9AView%20%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B/)
