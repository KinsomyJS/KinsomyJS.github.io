---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(七)Activity启动流程解析          # 标题 
subtitle:   Android advance #副标题
date:       2019-04-02            # 时间
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

    if (app.isolatedEntryPoint != null) {
                // This is an isolated process which should just call an entry point instead of
                // being bound to an application.
                thread.runIsolatedEntryPoint(app.isolatedEntryPoint, app.isolatedEntryPointArgs);
            } else if (app.instr != null) {
                thread.bindApplication(processName, appInfo, providers,
                        app.instr.mClass,
                        profilerInfo, app.instr.mArguments,
                        app.instr.mWatcher,
                        app.instr.mUiAutomationConnection, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            } else {
                thread.bindApplication(processName, appInfo, providers, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.persistent,
                        new Configuration(getGlobalConfiguration()), app.compat,
                        getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, isAutofillCompatEnabled);
            }
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

上面的过程最关键的就是`mStackSupervisor.attachApplicationLocked(app)`,ActivityStackSupervisor内部维护了大量的activity的记录栈，同时还有管理activity栈的ActivityStack实例，在该方法里面找到当前栈的栈顶activity。找到要启动的activity之后调用注释中标注的realStartActivityLocked方法。

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

realStartActivityLocked看名字显然就是我们要找的启动Activity的地方了，里面做了一些关键的操作：
```java
final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
//添加LaunchActivityItem的callback
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
        System.identityHashCode(r), r.info,
        mergedConfiguration.getGlobalConfiguration(),
        mergedConfiguration.getOverrideConfiguration(), r.compat,
        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
        profilerInfo
final ActivityLifecycleItem lifecycleItem;
if (andResume) {
    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
} else {
    lifecycleItem = PauseActivityItem.obtain();
}
clientTransaction.setLifecycleStateRequest(lifecycleIte
// 启动事务
mService.getLifecycleManager().scheduleTransaction(clientTransaction);
```

在ClientTransaction里面加入的是LaunchActivityItem对象，他通过obtain构造得到。然后通过上面最后一行代码启动这个事务，调用的ActivityManagerService的LifecycleManager的scheduleTransaction方法。

```java
class ClientLifecycleManager {
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        //调用ClientTransaction的schedule方法
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }

    ...
}
//ClientTransaction的schedule方法
private IApplicationThread mClient;
public void schedule() throws RemoteException {
        //调用IApplicationThread的scheduleTransaction方法
        mClient.scheduleTransaction(this);
    }

//最终实际调用了ActivityThread里面的该方法
public void executeTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        //关键代码
        getTransactionExecutor().execute(transaction);
        transaction.recycle();
    }
```
从上面的分析最后还是回到了ActivityThread里面，第一行preExecute做了一些在客户端调度事务需要预先执行的操作，关键代码在第二行execute，是TransactionExecutor线程池的执行。
```java
public class TransactionExecutor {
    /**
     * Resolve transaction.
     * First all callbacks will be executed in the order they appear in the list. If a callback
     * requires a certain pre- or post-execution state, the client will be transitioned accordingly.
     * Then the client will cycle to the final lifecycle state if provided. Otherwise, it will
     * either remain in the initial state, or last state needed by a callback.
     */
    public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();
        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

        executeCallbacks(transaction);

        executeLifecycleState(transaction);
        mPendingActions.clear();
        log("End resolving transaction");
    }
}

public void executeCallbacks(ClientTransaction transaction) {
    ...
    item.execute(mTransactionHandler, token, mPendingActions);
}

public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    //通知系统activityStart
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client);
    //调用ActivityThread的handleLaunchActivity
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```
上面在realStartActivityLocked方法里面我们调用addCallback传入的是LaunchActivityItem，这里excute方法里面执行了executeCallbacks方法，就会执行LaunchActivityItem的execute方法。方法内首先向系统发送"activityStart"通知，然后调用ClientTransactionHandler的handleLaunchActivity，因为ActivityThread类集成了ClientTransactionHandler，所以实际上还是调用了ActivityThread的handleLaunchActivity方法，handleLaunchActivity里面调用了performLaunchActivity创建要启动的Activity。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        //获得类加载器
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //Instrumentation构造Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
        ...

        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            Window window = null;
            appContext.setOuterContext(activity);
            //activity初始化
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback);
            ...
            //回调onCreate生命周期
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            r.activity = activity;
        }
        r.setState(ON_CREATE);
        mActivities.put(r.token, r);
   
    return activity;
}
```
performLaunchActivity通过类加载器委托mInstrumentation构造出要启动的Activity，然后做了Activity的相关初始化，最后调用callActivityOnCreate执行onCreate相关操作，回调onCreate生命周期，这样就完成了Activity的启动流程。

## 4 参考资料
* [Activity启动流程（下）](https://blog.csdn.net/mingyunxiaohai/article/details/88232680)

* [Android App启动流程](https://www.jianshu.com/p/49797b2ade02)
