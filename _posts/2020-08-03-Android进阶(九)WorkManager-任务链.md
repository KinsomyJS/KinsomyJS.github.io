---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(九)WorkManager-任务链          # 标题 
subtitle:   任务队列调度 #副标题
date:       2020-08-03            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 引言
上一篇文章[WorkManager解析](https://kinsomyjs.github.io/)对WorkManger最简单的使用案例做了介绍，同时对流程源码做了概括地分析，在文末提到了一个任务链的概念，可以使用WorkManager创建一个任务的链条，并将它们加入队列中，任务链指定多个依存任务并定义这些任务的运行顺序，这类似于RxJava的事件流的链式调用。这篇文章就来详细探讨其中的实现代码。

## 2 使用
任务链的使用我们看一段代码：
```kotlin
WorkManager.getInstance(myContext)
   .beginWith(listOf(req1, req2, req3))
   // 在前面三个任务都执行完成后才被执行
   .then(cache)
   .then(upload)
   .enqueue()
```

这段代码由三个任务按顺序链接在一起，首先先调用beginWith执行req1、req2、req3，他们可能并行运行，然后这三个任务的输出会合并传递给**cache**任务，cache执行完毕后再将输出传递到upload交由其执行。

## 3 源码解析
### 3.1 beginWith

来看一下beginWith的代码：
```java
public abstract class WorkManager {
    public final @NonNull WorkContinuation beginWith(@NonNull OneTimeWorkRequest work) {
        return beginWith(Collections.singletonList(work));
    }

    public abstract @NonNull WorkContinuation beginWith(@NonNull List<OneTimeWorkRequest> work);
}

public class WorkManagerImpl extends WorkManager {
    @Override
    public @NonNull WorkContinuation beginWith(@NonNull List<OneTimeWorkRequest> work) {
        if (work.isEmpty()) {
            throw new IllegalArgumentException(
                    "beginWith needs at least one OneTimeWorkRequest.");
        }
        return new WorkContinuationImpl(this, work);
    }
}
```
beginWith有两个重载方法，可以传入单个的OneTimeWorkRequest，也可以传入OneTimeWorkRequest的List集合，单个的OneTimeWorkRequest也会被包装成List，然后去调用WorkManagerImpl的beginWith，方法很简单，仅仅是用List构造了WorkContinuationImpl的实例，这个类上篇文章简单看过，现在继续深入。

### 3.2 WorkContinuationImpl
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

    WorkContinuationImpl(
            @NonNull WorkManagerImpl workManagerImpl,
            String name,
            ExistingWorkPolicy existingWorkPolicy,
            @NonNull List<? extends WorkRequest> work) {
        this(workManagerImpl, name, existingWorkPolicy, work, null);
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

    @Override
    public @NonNull WorkContinuation then(@NonNull List<OneTimeWorkRequest> work) {
        if (work.isEmpty()) {
            return this;
        } else {
            return new WorkContinuationImpl(mWorkManagerImpl,
                    mName,
                    ExistingWorkPolicy.KEEP,
                    work,
                    Collections.singletonList(this));
        }
    }
}
```

beginWith用的是第一个构造方法，因为是链的头部，所以最后一个参数parents为空，接着遍历work的List集合，将所有的work id加入到mAllIds中。

then方法同样是返回一个WorkContinuationImpl实例，它调用的是第三个构造方法，parents参数是其自身，将then之前已经加入mAllIds的work先导入进来，然后再将then的work导入，这样所有的work在mAllIds的集合里就是有序的。

### 3.3 WorkContinuationImpl#enqueue()
等到所有的worker都添加到队列中后，便可以调用enqueue方法去顺序执行任务：
```java 
public @NonNull Operation enqueue() {
        // Only enqueue if not already enqueued.
        if (!mEnqueued) {
            // The runnable walks the hierarchy of the continuations
            // and marks them enqueued using the markEnqueued() method, parent first.
            EnqueueRunnable runnable = new EnqueueRunnable(this);
            mWorkManagerImpl.getWorkTaskExecutor().executeOnBackgroundThread(runnable);
            mOperation = runnable.getOperation();
        } else {
            Logger.get().warning(TAG,
                    String.format("Already enqueued work ids (%s)", TextUtils.join(", ", mIds)));
        }
        return mOperation;
    }
```
这段代码也是上篇文章末尾提到的，首先构造了一个EnqueueRunnable实例，然后调用Executor去执行这个runnable实例，所以对于任务链的执行调度逻辑都在EnqueueRunnable中。

### 3.4 EnqueueRunnable
```java
public class EnqueueRunnable implements Runnable {
    @Override
    public void run() {
        try {
            //判断是否存在循环执行的任务,如果有抛出异常
            if (mWorkContinuation.hasCycles()) {
                throw new IllegalStateException(
                        String.format("WorkContinuation has cycles (%s)", mWorkContinuation));
            }
            boolean needsScheduling = addToDatabase();
            if (needsScheduling) {
                final Context context =
                        mWorkContinuation.getWorkManagerImpl().getApplicationContext();
                PackageManagerHelper.setComponentEnabled(context, RescheduleReceiver.class, true);
                scheduleWorkInBackground();
            }
            mOperation.setState(Operation.SUCCESS);
        } catch (Throwable exception) {
            mOperation.setState(new Operation.State.FAILURE(exception));
        }
    }

    @VisibleForTesting
    public boolean addToDatabase() {
        WorkManagerImpl workManagerImpl = mWorkContinuation.getWorkManagerImpl();
        WorkDatabase workDatabase = workManagerImpl.getWorkDatabase();
        workDatabase.beginTransaction();
        try {
            boolean needsScheduling = processContinuation(mWorkContinuation);
            workDatabase.setTransactionSuccessful();
            return needsScheduling;
        } finally {
            workDatabase.endTransaction();
        }
    }

    private static boolean processContinuation(@NonNull WorkContinuationImpl workContinuation) {
        boolean needsScheduling = false;
        List<WorkContinuationImpl> parents = workContinuation.getParents();
        if (parents != null) {
            for (WorkContinuationImpl parent : parents) {
                // When chaining off a completed continuation we need to pay
                // attention to parents that may have been marked as enqueued before.
                if (!parent.isEnqueued()) {
                    needsScheduling |= processContinuation(parent);
                } else {
                    Logger.get().warning(TAG, String.format("Already enqueued work ids (%s).",
                            TextUtils.join(", ", parent.getIds())));
                }
            }
        }
        needsScheduling |= enqueueContinuation(workContinuation);
        return needsScheduling;
    }

    private static boolean enqueueContinuation(@NonNull WorkContinuationImpl workContinuation) {
        Set<String> prerequisiteIds = WorkContinuationImpl.prerequisitesFor(workContinuation);

        boolean needsScheduling = enqueueWorkWithPrerequisites(
                workContinuation.getWorkManagerImpl(),
                workContinuation.getWork(),
                prerequisiteIds.toArray(new String[0]),
                workContinuation.getName(),
                workContinuation.getExistingWorkPolicy());

        workContinuation.markEnqueued();
        return needsScheduling;
    }
}
```
既然是一个Runnable实例，那么首先是要看run方法，run方法先去循环判断是否当前的任务id是否在它任务链上层被访问过，如果是，那就是存在循环重复执行的情况，就直接抛出异常；如果不存在上述情况，就调用addToDatabase方法，是将WorkSpec添加到数据库，然后调用processContinuation去检查是否有需要调度的任务，先获取当前workContinuation的parents，如果parents不为空并且没有被执行过，那就递归调用先去按序执行parent，执行的方法就是enqueueContinuation，这个方法调用了enqueueWorkWithPrerequisites，对任务进行排队，同时跟踪任务的先前任务，一旦找到需要执行的，needsScheduling会返回true，就把RescheduleReceiver的广播接收打开，然后调用scheduleWorkInBackground方法执行work，如果没有出现异常，将会在执行结束后将状态设为SUCCESS，否则在catch捕获异常时设为FAILURE。

### 3.5 scheduleWorkInBackground

```java
@VisibleForTesting
public void scheduleWorkInBackground() {
    WorkManagerImpl workManager = mWorkContinuation.getWorkManagerImpl();
    Schedulers.schedule(
            workManager.getConfiguration(),
            workManager.getWorkDatabase(),
            workManager.getSchedulers());
}
```
scheduleWorkInBackground调用了Schedulers.schedule方法：
```java
public static void schedule(
            @NonNull Configuration configuration,
            @NonNull WorkDatabase workDatabase,
            List<Scheduler> schedulers) {
        if (schedulers == null || schedulers.size() == 0) {
            return;
        }

        WorkSpecDao workSpecDao = workDatabase.workSpecDao();
        List<WorkSpec> eligibleWorkSpecsForLimitedSlots;
        List<WorkSpec> allEligibleWorkSpecs;

        workDatabase.beginTransaction();
        try {
            // 受调度限制的workSpec
            eligibleWorkSpecsForLimitedSlots = workSpecDao.getEligibleWorkForScheduling(
                    configuration.getMaxSchedulerLimit());

            // 不受限制的workSpec
            allEligibleWorkSpecs = workSpecDao.getAllEligibleWorkSpecsForScheduling();

            if (eligibleWorkSpecsForLimitedSlots != null
                    && eligibleWorkSpecsForLimitedSlots.size() > 0) {
                long now = System.currentTimeMillis();

                for (WorkSpec workSpec : eligibleWorkSpecsForLimitedSlots) {
                    workSpecDao.markWorkSpecScheduled(workSpec.id, now);
                }
            }
            workDatabase.setTransactionSuccessful();
        } finally {
            workDatabase.endTransaction();
        }

        if (eligibleWorkSpecsForLimitedSlots != null
                && eligibleWorkSpecsForLimitedSlots.size() > 0) {

            WorkSpec[] eligibleWorkSpecsArray =
                    new WorkSpec[eligibleWorkSpecsForLimitedSlots.size()];
            eligibleWorkSpecsArray =
                    eligibleWorkSpecsForLimitedSlots.toArray(eligibleWorkSpecsArray);

            // Delegate to the underlying schedulers.
            for (Scheduler scheduler : schedulers) {
                if (scheduler.hasLimitedSchedulingSlots()) {
                    scheduler.schedule(eligibleWorkSpecsArray);
                }
            }
        }

        if (allEligibleWorkSpecs != null && allEligibleWorkSpecs.size() > 0) {
            WorkSpec[] enqueuedWorkSpecsArray = new WorkSpec[allEligibleWorkSpecs.size()];
            enqueuedWorkSpecsArray = allEligibleWorkSpecs.toArray(enqueuedWorkSpecsArray);
            // Delegate to the underlying schedulers.
            for (Scheduler scheduler : schedulers) {
                if (!scheduler.hasLimitedSchedulingSlots()) {
                    scheduler.schedule(enqueuedWorkSpecsArray);
                }
            }
        }
    }
```
schedule方法首先将受调度限制（没有被调度过的）eligibleWorkSpecsForLimitedSlots和不受调度限制的allEligibleWorkSpecs集合找到，然后将eligibleWorkSpecsForLimitedSlots标记为被调度过。接着循环遍历eligibleWorkSpecsForLimitedSlots调用 scheduler.schedule(eligibleWorkSpecsArray)执行workSpec，然后同样的循环调度allEligibleWorkSpecs。
这里的schedule是个代理方法，实际上会去调用的是GreedyScheduler类的schedule方法。

```java
public class GreedyScheduler implements Scheduler, WorkConstraintsCallback, ExecutionListener {
@Override
    public void schedule(@NonNull WorkSpec... workSpecs) {
        if (mIsMainProcess == null) {
            mIsMainProcess = TextUtils.equals(mContext.getPackageName(), getProcessName());
        }

        if (!mIsMainProcess) {
            Logger.get().info(TAG, "Ignoring schedule request in non-main process");
            return;
        }

        registerExecutionListenerIfNeeded();

        // Keep track of the list of new WorkSpecs whose constraints need to be tracked.
        // Add them to the known list of constrained WorkSpecs and call replace() on
        // WorkConstraintsTracker. That way we only need to synchronize on the part where we
        // are updating mConstrainedWorkSpecs.
        Set<WorkSpec> constrainedWorkSpecs = new HashSet<>();
        Set<String> constrainedWorkSpecIds = new HashSet<>();

        for (WorkSpec workSpec : workSpecs) {
            long nextRunTime = workSpec.calculateNextRunTime();
            long now = System.currentTimeMillis();
            if (workSpec.state == WorkInfo.State.ENQUEUED) {
                if (now < nextRunTime) {
                    // Future work
                    if (mDelayedWorkTracker != null) {
                        mDelayedWorkTracker.schedule(workSpec);
                    }
                } else if (workSpec.hasConstraints()) {
                    if (SDK_INT >= 23 && workSpec.constraints.requiresDeviceIdle()) {
                        // Ignore requests that have an idle mode constraint.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires device idle.",
                                        workSpec));
                    } else if (SDK_INT >= 24 && workSpec.constraints.hasContentUriTriggers()) {
                        // Ignore requests that have content uri triggers.
                        Logger.get().debug(TAG,
                                String.format("Ignoring WorkSpec %s, Requires ContentUri triggers.",
                                        workSpec));
                    } else {
                        constrainedWorkSpecs.add(workSpec);
                        constrainedWorkSpecIds.add(workSpec.id);
                    }
                } else {
                    Logger.get().debug(TAG, String.format("Starting work for %s", workSpec.id));
                    mWorkManagerImpl.startWork(workSpec.id);
                }
            }
        }

        // onExecuted() which is called on the main thread also modifies the list of mConstrained
        // WorkSpecs. Therefore we need to lock here.
        synchronized (mLock) {
            if (!constrainedWorkSpecs.isEmpty()) {
                Logger.get().debug(TAG, String.format("Starting tracking for [%s]",
                        TextUtils.join(",", constrainedWorkSpecIds)));
                mConstrainedWorkSpecs.addAll(constrainedWorkSpecs);
                mWorkConstraintsTracker.replace(mConstrainedWorkSpecs);
            }
        }
    }
}
```
 * 1.首先判断是否在主进程，如果不是在主进程执行的调度任务，则直接return，忽略掉该请求。
 * 2.遍历方法参数传入的workSpecs数组，计算每个workSpecs的下一次可以被执行时间nextRunTime
 * 3.如果当前时间早于nextRunTime，就把他加入到mDelayedWorkTracker中延迟执行
 * 4.如果workSpec有限制则根据sdk版本做不同的处理
 * 5.如果上述都不符合，则可以开始任务，调用mWorkManagerImpl.startWork(workSpec.id)

```java 
 public boolean startWork(
            @NonNull String id,
            @Nullable WorkerParameters.RuntimeExtras runtimeExtras) {

        WorkerWrapper workWrapper;
        synchronized (mLock) {
            // Work may get triggered multiple times if they have passing constraints
            // and new work with those constraints are added.
            if (isEnqueued(id)) {
                Logger.get().debug(
                        TAG,
                        String.format("Work %s is already enqueued for processing", id));
                return false;
            }

            workWrapper =
                    new WorkerWrapper.Builder(
                            mAppContext,
                            mConfiguration,
                            mWorkTaskExecutor,
                            this,
                            mWorkDatabase,
                            id)
                            .withSchedulers(mSchedulers)
                            .withRuntimeExtras(runtimeExtras)
                            .build();
            ListenableFuture<Boolean> future = workWrapper.getFuture();
            future.addListener(
                    new FutureListener(this, id, future),
                    mWorkTaskExecutor.getMainThreadExecutor());
            mEnqueuedWorkMap.put(id, workWrapper);
        }
        mWorkTaskExecutor.getBackgroundExecutor().execute(workWrapper);
        Logger.get().debug(TAG, String.format("%s: processing %s", getClass().getSimpleName(), id));
        return true;
    }

```
跟着startWork方法一直追进去，会找到Processor的startWork方法，work被WorkerWrapper包装了起来，WorkerWrapper是一个runnable,mWorkTaskExecutor.getBackgroundExecutor().execute(workWrapper)去执行这个runnable。

进到WorkerWrapper的run()方法里去看,在run里面回去调用worker的startWork方法，这个方法会执行worker的doWork方法，也就是我们自己实现的doWork逻辑，同时会监听Future的回调，把任务结果result回调回去。
```java 
public void run() {
    mTags = mWorkTagDao.getTagsForWorkSpecId(mWorkSpecId);
    mWorkDescription = createWorkDescription(mTags);
    runWorker();
}

private void runWorker() {
    ...
    if (trySetRunning()) {
            if (tryCheckForInterruptionAndResolve()) {
                return;
            }

            final SettableFuture<ListenableWorker.Result> future = SettableFuture.create();
            // Call mWorker.startWork() on the main thread.
            mWorkTaskExecutor.getMainThreadExecutor()
                    .execute(new Runnable() {
                        @Override
                        public void run() {
                            try {
                                Logger.get().debug(TAG, String.format("Starting work for %s",
                                        mWorkSpec.workerClassName));
                                mInnerFuture = mWorker.startWork();
                                future.setFuture(mInnerFuture);
                            } catch (Throwable e) {
                                future.setException(e);
                            }

                        }
                    });
            final String workDescription = mWorkDescription;
            future.addListener(new Runnable() {
                @Override
                @SuppressLint("SyntheticAccessor")
                public void run() {
                    try {
                        // If the ListenableWorker returns a null result treat it as a failure.
                        ListenableWorker.Result result = future.get();
                        if (result == null) {
                            Logger.get().error(TAG, String.format(
                                    "%s returned a null result. Treating it as a failure.",
                                    mWorkSpec.workerClassName));
                        } else {
                            Logger.get().debug(TAG, String.format("%s returned a %s result.",
                                    mWorkSpec.workerClassName, result));
                            mResult = result;
                        }
                    } catch (CancellationException exception) {
                        // Cancellations need to be treated with care here because innerFuture
                        // cancellations will bubble up, and we need to gracefully handle that.
                        Logger.get().info(TAG, String.format("%s was cancelled", workDescription),
                                exception);
                    } catch (InterruptedException | ExecutionException exception) {
                        Logger.get().error(TAG,
                                String.format("%s failed because it threw an exception/error",
                                        workDescription), exception);
                    } finally {
                        onWorkFinished();
                    }
                }
            }, mWorkTaskExecutor.getBackgroundExecutor());
        ...
    }
    ...
}
```
这里的代码还是很复杂的，doWork方法的调用逻辑很深，并且要完全读懂里面的各种逻辑还需要很多时间。