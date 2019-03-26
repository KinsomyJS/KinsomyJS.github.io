---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(七)AMS解析          # 标题 
subtitle:   Android advance #副标题
date:       2019-03-21            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 前言
ActivityThread在Application进程中管理着主线程的执行，同时根据Activity Manager Service（AMS）的请求负责调度和执行activities，broadcasts和其他操作。

ActivityThread作为app的入口，启动一个app时会先执行里面的main()方法，这个逻辑和一个普通的java程序相同。通过分析ActivityThread的源码来了解app的启动流程。

## 2 ActivityThread
先来看一下main方法的代码,为了看起来方便，只保留了一些关键代码：
```java
public static void main(String[] args) {
        
        ...

        Looper.prepareMainLooper();
        //如果main方法参数不为空，且从命令行启动，那么startSep则为114
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

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

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
### 2.1 初始化Looper
main方法首先会调用`Looper.prepareMainLooper();`,这个方法是在app启动时会主线程初始化一个`Looper`,是不是很像我们在异步线程里面调用的`Looper.prepare()`方法，当我们在异步线程想要用handler来处理消息时，必须要先调用Looper.prepare()获得消息循环，而在主线程则不需要调用该方法，就是因为在app启动调用main()方法时就已经帮你创建好了主线程的looper。

对比看一下两个方法，原理就是在当前线程的threadlocal里面set一个looper对象：
```java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

然后在main()的最后会调用`Looper.loop()`开启消息循环，从消息队列里取消息。这一块是Handler机制，这篇文章不做详细讲解。

### 2.2 attach
```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                ensureJitEnabled();
            }
        });
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        //拿到ActivityManagerService
        final IActivityManager mgr = ActivityManager.getService();
        try {
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
        ...
    }
    ...
}
```
attch方法接受两个参数，第一个bool值是判断是否启动的是系统应用，第二个是启动序号，只需要看非系统应用部分即可，所以删除判断为system参数为true的分支。
首先先通过getService方法拿到IActivityManager对象，这是一个接口，ActivityManagerService继承了它，因此拿到的实际是ActivityManagerService的实例。
```java
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    //通过binder机制拿到IBinder实例
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    //调用asInterface跨进程拿到IActivityManager
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

接着调用ActivityManagerService的attachApplication方法，这个方法显然就是将当前要启动的Applicationthread和ActivityManagerService所在的system_server进程绑定。下面就进入了ActivityManagerService的分析。

## 3 ActivityManagerService
上面调用的attachApplication源码如下：
```java
public final void attachApplication(IApplicationThread thread, long startSeq) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid, callingUid, startSeq);
        //保存IPC调用的id
        Binder.restoreCallingIdentity(origId);
    }
}
```
关键代码在于attachApplicationLocked的调用：
```java
private final boolean attachApplicationLocked(IApplicationThread thread,
        int pid, int callingUid, long startSeq) {
    //先找到被attach的进程记录
    ProcessRecord app;
    long startTime = SystemClock.uptimeMillis();
    //如果调用的进程id不是system_server且大于等于0
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }
    //可能进程调用在更新内部状态之前
    if (app == null && startSeq > 0) {
        final ProcessRecord pending = mPendingStarts.get(startSeq);
        if (pending != null && pending.startUid == callingUid
                && handleProcessStartedLocked(pending, pid, pending.usingWrapper,
                        startSeq, true)) {
            app = pending;
        }
    }
    ...
    //配置启动信息 检查各种状态
    ...
    //如果正常启动模式
    if (normalMode) {
        try {
          if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }
    //如果上面没有抛出异常则获取应该运行在该进程上的服务
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
            checkTime(startTime, "attachApplicationLocked: after mServices.attachApplicationLocked");
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
            badApp = true;
        }
    }
    //检查进程是否有广播接收器
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
            checkTime(startTime, "attachApplicationLocked: after sendPendingBroadcastsLocked");
        } catch (Exception e) {
            // If the app died trying to launch the receiver we declare it 'bad'
            Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
            badApp = true;
        }
    }
}
```

上面的过程最关键的就是`mStackSupervisor.attachApplicationLocked(app)`,ActivityStackSupervisor内部维护了大量的activity的记录栈，同时还有管理activity栈的ActivityStack实例。
```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    //获得app的进程名
    final String processName = app.processName;
    boolean didSomething = false;
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        final ActivityDisplay display = mActivityDisplays.valueAt(displayNdx);
        //找到当前进程的activity栈
        for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = display.getChildAt(stackNdx);
            if (!isFocusedStack(stack)) {
                continue;
            }
            stack.getAllRunningVisibleActivitiesLocked(mTmpActivityList);
            final ActivityRecord top = stack.topRunningActivityLocked();
            final int size = mTmpActivityList.size();
            for (int i = 0; i < size; i++) {
                final ActivityRecord activity = mTmpActivityList.get(i);
                if (activity.app == null && app.uid == activity.info.applicationInfo.uid
                        && processName.equals(activity.processName)) {
                    try {
                        //启动activity
                        if (realStartActivityLocked(activity, app,
                                top == activity /* andResume */, true /* checkConfig */)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Exception in new application when starting activity "
                                + top.intent.getComponent().flattenToShortString(), e);
                        throw e;
                    }
                }
            }
        }
    }
    if (!didSomething) {
        ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
    }
    return didSomething;
}
```

