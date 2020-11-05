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



