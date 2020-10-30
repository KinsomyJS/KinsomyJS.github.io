---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(八)WorkManager解析          # 标题 
subtitle:   Android advance #副标题
date:       2020-08-01            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 引言
WorkManager是Android Jetpack的一个库，可以在达到约束条件的情况下执行可延期的有保证性的后台任务，适用于即便在应用退出后也要能保证运行的工作任务，是后台任务框架的优秀解决方案。

WorkManager又不完全是一个全新的后台任务框架，它的底层依然使用了Android之前的API，会根据设备的版本去选择调用调用JobScheduler或者Firebase JobDispatcher,或者AlarmManager来执行任务。上层封装了一致性API，让开发者不用再为设备版本不同去考虑不同的后台任务方案。

![](https://pic2.zhimg.com/80/v2-99d2d71bb866c97bb8224b3b7179c7b1_720w.jpg)

除此之外，WorkManager还具备很多优点：
* 工作约束
* 强大的调度
* 灵活的重试策略
* 任务链接，按照需要依次执行
* 内置线程互操作性，无缝集成RxJava和协程

具体介绍查看[官方文档](https://developer.android.com/topic/libraries/architecture/workmanager)

## 2 使用
开始使用WorkManager前，先添加如下依赖,WorkManager支持Java和kotlin，同时还增加了对协程和RxJava的支持。
```gradle
dependencies {
  def work_version = "2.4.0"

    // (Java only)
    implementation "androidx.work:work-runtime:$work_version"

    // Kotlin + coroutines
    implementation "androidx.work:work-runtime-ktx:$work_version"

    // optional - RxJava2 support
    implementation "androidx.work:work-rxjava2:$work_version"

    // optional - GCMNetworkManager support
    implementation "androidx.work:work-gcm:$work_version"

    // optional - Test helpers
    androidTestImplementation "androidx.work:work-testing:$work_version"
  }
```

### 2.1 定义任务Worker
首先先要定义一个执行任务，这里定义一个循环执行上传图片的任务。
```kotlin
import androidx.work.Worker
import androidx.work.WorkerParameters

class UploadWorker(
    val context: Context, workerParams: WorkerParameters
) : Worker(context, workerParams) {
    override fun doWork(): Result {
        uploadImg()
        return Result.success();
    }

    fun uploadImg() {
        while (true) {
            // 模拟上传耗时
            Thread.sleep(1000)
            Log.d("workermanager", "uploadImg...")
        }
    }
}
```
定义一个任务都需要继承WorkManager的抽象类Worker，并且重写doWork()方法，这里是你要执行的操作代码，doWork()返回的 Result 会通知 WorkManager 服务工作是否成功，以及工作失败时是否应重试工作，有三种：
* Result.success() 任务执行成功
* Result.failure() 任务执行失败
* Result.retry() 任务执行失败，按照重试策略进行重试

### 2.2 创建WorkRequest
```kotlin
// 单次任务
val req1: WorkRequest = OneTimeWorkRequest.from(UploadWorker::class.java)

// 周期任务，最短间隔15分钟
val req2 = PeriodicWorkRequestBuilder<UploadWorker>(15, TimeUnit.MINUTES).build()

// 增加约束，仅在用户设备正在充电且连接到 Wi-Fi 网络时才会运行
val constraints = Constraints.Builder()
   .setRequiredNetworkType(NetworkType.UNMETERED)
   .setRequiresCharging(true)
   .build()

val req3: WorkRequest = OneTimeWorkRequestBuilder<MyWork>()
        .setInitialDelay(10, TimeUnit.MINUTES) // 延迟10分钟
        .setConstraints(constraints)
        .build()
```
还有更多的创建WorkRequest也可以查阅官方文档，这里只给出最简单的例子，抓住主线，为后面解析代码做准备。

### 2.3 执行WorkRequest
```kotlin
WorkManager.getInstance(mContext).enqueue(request)
// 执行唯一任务
WorkManager.getInstance(mContext).enqueueUniqueWork(uniqueWorkName,existingWorkPolicy,request)
// 定期执行唯一任务
WorkManager.getInstance(mContext).enqueueUniquePeriodicWork(uniqueWorkName,existingWorkPolicy,request)
```

## 3 源码解析
上面介绍了一个最简单的从创建任务到执行的过程，下面就沿着这条主线来分析源代码。

### 3.1 Worker 
一次性Worker的生命周期如图所示：
![](https://pic1.zhimg.com/80/v2-f222113b1f81629f3e891ac4b3106aa8_720w.jpg)

定期Worker的生命周期如图缩视：
![](https://pic1.zhimg.com/80/v2-fd807425854dae019e6254347b7f94a0_720w.jpg)

```java
public abstract class Worker extends ListenableWorker {

    // Package-private to avoid synthetic accessor.
    SettableFuture<Result> mFuture;

    @Keep
    @SuppressLint("BanKeepAnnotation")
    public Worker(@NonNull Context context, @NonNull WorkerParameters workerParams) {
        super(context, workerParams);
    }

    @WorkerThread
    public abstract @NonNull Result doWork();

    @Override
    public final @NonNull ListenableFuture<Result> startWork() {
        mFuture = SettableFuture.create();
        getBackgroundExecutor().execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Result result = doWork();
                    mFuture.set(result);
                } catch (Throwable throwable) {
                    mFuture.setException(throwable);
                }
            }
        });
        return mFuture;
    }
}
```
Worker是一个抽象类，所有创建的任务都需要继承它，Worker又继承ListenableWorker，构造函数有两个参数，第二个参数WorkerParameters是这个WorkRequest的一些参数。

```java 
public final class WorkerParameters {

    private @NonNull UUID mId; // 唯一ID
    private @NonNull Data mInputData; // 请求的输入数据
    private @NonNull Set<String> mTags; // 该请求的标签
    private @NonNull RuntimeExtras mRuntimeExtras;
    private int mRunAttemptCount;
    private @NonNull Executor mBackgroundExecutor;
    private @NonNull TaskExecutor mWorkTaskExecutor;
    private @NonNull WorkerFactory mWorkerFactory;
    private @NonNull ProgressUpdater mProgressUpdater;
    private @NonNull ForegroundUpdater mForegroundUpdater;
}
```

Worker继承ListenableWorker并重载了startWork()方法，ListenableWorker在运行时被WorkerFactory初始化，这个方法在主线程调用被调用，startWork()从后台线程池获得Executor然后执行doWork()方法，并将doWork()返回的result回调出去供开发者监听。所以每次自定义Worker时，需要做的就是重写doWork()写上自己的实现。

### 3.2 WorkRequest
WorkRequest是一个抽象基类，它承载了定义好的Work，然后给Work添加一些定制化的参数，例如延时执行，循环执行，重试策略等，包装好之后就被丢到WorkManager里面执行。

WorkRequest有两个实现类，分别是OneTimeWorkRequest（单次任务请求）和PeriodicWorkRequest（重复任务请求）。

先来看一下基类WorkRequest的主要代码：
```java
public abstract class WorkRequest {
    // 默认重试退避时间
    public static final long DEFAULT_BACKOFF_DELAY_MILLIS = 30000L;
    // 最大重试退避时间 5小时
    public static final long MAX_BACKOFF_MILLIS = 5 * 60 * 60 * 1000;
    // 最小重试退避时间 10秒钟
    public static final long MIN_BACKOFF_MILLIS = 10 * 1000;
    /**
     * @hide
     */
    // 隐藏方法，不对外暴露，要使用就用他的实现类提供的方法，该构造方法要三个参数：
    // id:很好理解，就是标记request的唯一标识符
    // workSpec：存储work信息的逻辑单元，保存了request对应的work的信息
    // tags: WorkRequest可以调用addTag方法增加一个标记，可以通过workManager的xxxByTag系列函数来获取一些信息
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    protected WorkRequest(@NonNull UUID id, @NonNull WorkSpec workSpec, @NonNull Set<String> tags) {
        mId = id;
        mWorkSpec = workSpec;
        mTags = tags;
    }

    /**
     * Builder建造类，用来定制化workRequest
     */
    public abstract static class Builder<B extends Builder<?, ?>, W extends WorkRequest> {

        boolean mBackoffCriteriaSet = false;
        UUID mId;
        WorkSpec mWorkSpec;
        Set<String> mTags = new HashSet<>();
        Class<? extends ListenableWorker> mWorkerClass;

        Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
            mId = UUID.randomUUID();
            mWorkerClass = workerClass;
            mWorkSpec = new WorkSpec(mId.toString(), workerClass.getName());
            addTag(workerClass.getName());
        }
        ......
    }
}
```
接着再来看一下OneTimeWorkRequest的具体实现：

提供了两个快速创建OneTimeWorkRequest的from方法，from(workerClass)是传入一个worker， from(workerClasses) 则是可以传入list的worker，可以同时创建多个任务。

另外就是继承了WorkRequest的Builder类，定制化的创建OneTimeWorkRequest。

整个类非常简单，看看源码就能理解了，因为本身就是一个构造请求的类，往里面加各种参数。

```java
public final class OneTimeWorkRequest extends WorkRequest {

    public static @NonNull OneTimeWorkRequest from(
            @NonNull Class<? extends ListenableWorker> workerClass) {
        return new OneTimeWorkRequest.Builder(workerClass).build();
    }

    public static @NonNull List<OneTimeWorkRequest> from(
            @NonNull List<Class<? extends ListenableWorker>> workerClasses) {
        List<OneTimeWorkRequest> workList = new ArrayList<>(workerClasses.size());
        for (Class<? extends ListenableWorker> workerClass : workerClasses) {
            workList.add(new OneTimeWorkRequest.Builder(workerClass).build());
        }
        return workList;
    }

    OneTimeWorkRequest(Builder builder) {
        super(builder.mId, builder.mWorkSpec, builder.mTags);
    }

    /**
     * Builder for {@link OneTimeWorkRequest}s.
     */
    public static final class Builder extends WorkRequest.Builder<Builder, OneTimeWorkRequest> {

        /**
         * Creates a {@link OneTimeWorkRequest}.
         *
         * @param workerClass The {@link ListenableWorker} class to run for this work
         */
        public Builder(@NonNull Class<? extends ListenableWorker> workerClass) {
            super(workerClass);
            mWorkSpec.inputMergerClassName = OverwritingInputMerger.class.getName();
        }

        public @NonNull Builder setInputMerger(@NonNull Class<? extends InputMerger> inputMerger) {
            mWorkSpec.inputMergerClassName = inputMerger.getName();
            return this;
        }

        @Override
        @NonNull OneTimeWorkRequest buildInternal() {
            if (mBackoffCriteriaSet
                    && Build.VERSION.SDK_INT >= 23
                    && mWorkSpec.constraints.requiresDeviceIdle()) {
                throw new IllegalArgumentException(
                        "Cannot set backoff criteria on an idle mode job");
            }
            if (mWorkSpec.runInForeground
                    && Build.VERSION.SDK_INT >= 23
                    && mWorkSpec.constraints.requiresDeviceIdle()) {
                throw new IllegalArgumentException(
                        "Cannot run in foreground with an idle mode constraint");
            }
            return new OneTimeWorkRequest(this);
        }

        @Override
        @NonNull Builder getThis() {
            return this;
        }
    }
}
```

### 3.3 WorkManager
WorkManager也是一个抽象基类，它的实现类是WorkManagerImpl，WorkManager看名字就知道，对WorkRequest进行管理，包括执行、控制、取消、获取信息、生成观察者等功能，下面来看看其中比较常用的api源码，看看这些功能是怎么实现的。
```java 
public abstract class WorkManager {
    public static @NonNull WorkManager getInstance(@NonNull Context context) {
        return WorkManagerImpl.getInstance(context);
    }

    public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {
        WorkManagerImpl.initialize(context, configuration);
    }
    ....
}
```
WorkManager的单例方法getInstance实际上调用了WorkManagerImpl的getInstance方法，剩下的代码也都是一些冲向方法，在WorkManagerImpl中会被实现，所以将关注的重点放在WorkManagerImpl类上,下面的代码做了精简。

```java
public class WorkManagerImpl extends WorkManager {
    public static @Nullable WorkManagerImpl getInstance() {
        synchronized (sLock) {
            if (sDelegatedInstance != null) {
                return sDelegatedInstance;
            }
            return sDefaultInstance;
        }
    }
    // 单例构造WorkManagerImpl实例对象
    @RestrictTo(RestrictTo.Scope.LIBRARY_GROUP)
    public static @NonNull WorkManagerImpl getInstance(@NonNull Context context) {
        synchronized (sLock) {
            WorkManagerImpl instance = getInstance();
            if (instance == null) {
                Context appContext = context.getApplicationContext();
                if (appContext instanceof Configuration.Provider) {
                    initialize(
                            appContext,
                            ((Configuration.Provider) appContext).getWorkManagerConfiguration());
                    instance = getInstance(appContext);
                } else {
                    throw new IllegalStateException("...");
                }
            }
            return instance;
        }
    }
}
```

我们断点进到getInstance内部发现在第一次的时候**sDelegatedInstance**和**sDefaultInstance**就已经被创建出来了，深入追溯代码，在WorkManagerImpl里面有一个initialize的静态方法：
```java
public static void initialize(@NonNull Context context, @NonNull Configuration configuration) {
        synchronized (sLock) {
            if (sDelegatedInstance != null && sDefaultInstance != null) {
                throw new IllegalStateException("...");
            }

            if (sDelegatedInstance == null) {
                context = context.getApplicationContext();
                if (sDefaultInstance == null) {
                    sDefaultInstance = new WorkManagerImpl(
                            context,
                            configuration,
                            new WorkManagerTaskExecutor(configuration.getTaskExecutor()));
                }
                sDelegatedInstance = sDefaultInstance;
            }
        }
    }
```
initialize内sDefaultInstance被构造出来，同时赋值给sDelegatedInstance，该initialize又被基类WorkManager的initialize方法调用，WorkManager的initialize方法被WorkManagerInitializer类的onCreate()生命周期方法调用，WorkManagerInitializer类是什么，追进去发现，它是一个ContentProvider，并且在work-runtime类库的AndroidManifest.xml中被声明，所以在在我们第一次进到getInstance方法中的时候，单例实例就已经存在。
```java
public class WorkManagerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        // Initialize WorkManager with the default configuration.
        WorkManager.initialize(getContext(), new Configuration.Builder().build());
        return true;
    }
    ...
}
```

```xml
<application>
    <provider
        android:name="androidx.work.impl.WorkManagerInitializer"
        android:authorities="${applicationId}.workmanager-init"
        android:directBootAware="false"
        android:exported="false"
        android:multiprocess="true"
        tools:targetApi="n" />
</application>
```

上面讲到，initialize方法里调用了WorkManagerImpl的构造方法，下面就看看构造方法的具体代码：
```java
sDefaultInstance = new WorkManagerImpl(
                            context,
                            configuration,
                            new WorkManagerTaskExecutor(configuration.getTaskExecutor()));

public WorkManagerImpl(
            @NonNull Context context,
            @NonNull Configuration configuration,
            @NonNull TaskExecutor workTaskExecutor) {
        this(context,
                configuration,
                workTaskExecutor,
                context.getResources().getBoolean(R.bool.workmanager_test_configuration));
    }
```
有多种构造函数，我们选择参数最少的一个，第一个参数是上下文，第二个就是配置项，这里就是用的默认配置类Configuration，第三个参数是TaskExecutor，看上去就是一个线程池，使用的是java.util.concurrent包下面的Executor类，很容易联想到后面的方案，就是用线程池调度去执行任务。

下面就来看最后的执行步骤enqueue方法：
```java
// WorkManager.java
@NonNull
public final Operation enqueue(@NonNull WorkRequest workRequest) {
    return enqueue(Collections.singletonList(workRequest));
}

// WorkManagerImpl.java
public Operation enqueue(
            @NonNull List<? extends WorkRequest> workRequests) {
        if (workRequests.isEmpty()) {
            throw new IllegalArgumentException(
                    "enqueue needs at least one WorkRequest.");
        }
        return new WorkContinuationImpl(this, workRequests).enqueue();
    }
```
先是调用WorkManager的enqueue方法，然后调用WorkManagerImpl的enqueue方法，同时将单个WorkRequest包裹成List传进去，然后new一个WorkContinuationImpl类的实例，并且调用其enqueue方法，追进去看WorkContinuationImpl代码：
```java
public class WorkContinuationImpl extends WorkContinuation {
    WorkContinuationImpl(
            @NonNull WorkManagerImpl workManagerImpl,
            @NonNull List<? extends WorkRequest> work) {
        this(
                workManagerImpl,
                null,
                ExistingWorkPolicy.KEEP,
                work,
                null);
    }
    WorkContinuationImpl(@NonNull WorkManagerImpl workManagerImpl,
            String name,
            ExistingWorkPolicy existingWorkPolicy,
            @NonNull List<? extends WorkRequest> work,
            @Nullable List<WorkContinuationImpl> parents) {
        mWorkManagerImpl = workManagerImpl;
        mName = name;
        mExistingWorkPolicy = existingWorkPolicy;
        mWork = work;
        mParents = parents;
        mIds = new ArrayList<>(mWork.size());
        mAllIds = new ArrayList<>();
        if (parents != null) {
            for (WorkContinuationImpl parent : parents) {
                mAllIds.addAll(parent.mAllIds);
            }
        }
        for (int i = 0; i < work.size(); i++) {
            String id = work.get(i).getStringId();
            mIds.add(id);
            mAllIds.add(id);
        }
    }
    public @NonNull Operation enqueue() {
        if (!mEnqueued) {
            EnqueueRunnable runnable = new EnqueueRunnable(this);
            mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
            mOperation = runnable.getOperation();
        } else {
            Logger.get().warning(TAG,
                    String.format("Already enqueued work ids (%s)", TextUtils.join(", ", mIds)));
        }
        return mOperation;
    }
}
```
构造方法就是将传进来的WorkRequest的id添加到集合中，方便后面控制，然后enqueue代码就很熟悉，类似线程池操作，

## 4 总结
到此为止，一个最简单的WorkManager流程就分析结束了，但是这个库的功能非常强大，后面还会针对详细的特点进行解析，例如任务链的调度，应用退出再进入还能继续执行上次的任务，以及结合LiveData等等。

## 5 参考资料
* [官方文档](https://developer.android.com/topic/libraries/architecture/workmanager)



