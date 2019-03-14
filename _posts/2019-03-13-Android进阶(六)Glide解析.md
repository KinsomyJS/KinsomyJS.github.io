---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(六)Glide解析-加载流程          # 标题 
subtitle:   Android advance #副标题
date:       2019-03-13            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 引言
一直想要阅读Glide源码，但是苦于时间和功力都不够，总是断断续续的，趁着现在有一些空暇时间，来简要分析Glide的源码。Glide的实现太过复杂，不可能做到面面俱到，如果每一行都细致分析，很容易陷入作者的优化细节中去而偏离主线，因此只针对几个主要功能做解析即可。
以下分析全部基于Glide v4.9.0。

## 2 初始化
Glide最常见的用法就是如下一行代码：
```java
Glide.with(context).load(url).into(imageView);
```
一步步来分析如何将url图片加载到imageview上来。
### 2.1 with
```java
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}
public static RequestManager with(@NonNull Activity activity) {
  return getRetriever(activity).get(activity);
}
public static RequestManager with(@NonNull FragmentActivity activity) {
  return getRetriever(activity).get(activity);
}
public static RequestManager with(@NonNull Fragment fragment) {
  return getRetriever(fragment.getActivity()).get(fragment);
}
public static RequestManager with(@NonNull View view) {
  return getRetriever(view.getContext()).get(view);
}
```
with方法是Glide类中的一组同名static重载函数，可以传入多种上下文，方法体内是调用`getRetriever`获得`RequestManagerRetriever`实例对象，再调用其get方法返回一个`RequestManager`实例。
```java
private static volatile Glide glide;

private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
  ...
  return Glide.get(context).getRequestManagerRetriever();
}

public static Glide get(@NonNull Context context) {
  if (glide == null) {
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context);
      }
    }
  }
  return glide;
}

public RequestManagerRetriever getRequestManagerRetriever() {
  return requestManagerRetriever;
}
```
这里Glide的get方法用了DCL单例，然后拿到Glide的成员变量requestManagerRetriever。
然后再看RequestManagerRetriever类。
```java
  @NonNull
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }

    return getApplicationManager(context);
  }
```
这是上文with方法里调用的get方法，这里会对传入的context做判断，如果方法调用是在主线程同时context不是Application，则会根据context的类型调用一组重载的get方法，否则就调用getApplicationManager。

那这两个分支有什么区别呢？具体看一下两个处理方法。
```java
private RequestManager getApplicationManager(@NonNull Context context) {
  // Either an application context or we're on a background thread.
  if (applicationManager == null) {
    synchronized (this) {
      if (applicationManager == null) {
        // Normally pause/resume is taken care of by the fragment we add to the fragment or
        // activity. However, in this case since the manager attached to the application will not
        // receive lifecycle events, we must force the manager to start resumed using
        // ApplicationLifecycle.
        // TODO(b/27524013): Factor out this Glide.get() call.
        Glide glide = Glide.get(context.getApplicationContext());
        applicationManager =
            factory.build(
                glide,
                new ApplicationLifecycle(),
                new EmptyRequestManagerTreeNode(),
                context.getApplicationContext());
      }
    }
  }
  return applicationManager;
}
public RequestManager get(@NonNull FragmentActivity activity) {
  if (Util.isOnBackgroundThread()) {
    return get(activity.getApplicationContext());
  } else {
    assertNotDestroyed(activity);
    FragmentManager fm = activity.getSupportFragmentManager();
    return supportFragmentGet(
        activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
  }
}

private SupportRequestManagerFragment getSupportRequestManagerFragment(
    @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
  SupportRequestManagerFragment current =
      (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
  if (current == null) {
    current = pendingSupportRequestManagerFragments.get(fm);
    if (current == null) {
      current = new SupportRequestManagerFragment();
      current.setParentFragmentHint(parentHint);
      if (isParentVisible) {
        current.getGlideLifecycle().onStart();
      }
      pendingSupportRequestManagerFragments.put(fm, current);
      fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
      handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
    }
  }
  return current;
}
```
这里的区别主要是将glide和不同的上下文的声明周期绑定，如果是Application或者不在主线程调用，那requetmanager的生命周期和Application相关，否则则会和当前页面的fragmentManager的声明周期相关。因为Activity下fragmentManager的生命周期和Activity相同。所以不管是Activity还是fragment，最后都会委托给fragmentManager做生命周期的管理。

在getSupportRequestManagerFragment方法中可以看到如果activity下的fragmentmanager没有找到tag为FRAGMENT_TAG的fragment，就会创建一个隐藏的fragment，然后添加到fragmentmanager内。

总结来说with方法的作用就是获得当前上下文，构造出和上下文生命周期绑定的requestmanager，自动管理glide的加载开始和停止。

### 2.2 load
load方法也是一组重载方法，定义在`interface ModelTypes<T>`接口里，这是一个泛型接口，规定了load想要返回的数据类型，`RequestManager`类实现了该接口，泛型为Drawable类。
```java
public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);
}
public RequestBuilder<Drawable> load(@Nullable File file) {
  return asDrawable().load(file);
}
public RequestBuilder<Drawable> load(@Nullable Uri uri) {
  return asDrawable().load(uri);
}
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {
  return asDrawable().load(drawable);
}
public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) {
  return asDrawable().load(bitmap);
}
public RequestBuilder<Drawable> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
  return asDrawable().load(resourceId);
}
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
......
```
RequestManager下的load方法都返回RequestBuilder对象，显然是一个建造者模式，用来构建需要的属性。asDrawable方法调用的as方法实际上是调用RequestBuilder的构造方法。然后掉用RequestBuilder的load将需要加载的object传递给builder，然后load方法都会调用loadGeneric将不同的参数类型统一传给Object类的model成员变量，如果传递的参数类型是Drawable或者Bitmap，那么将会额外调用`.apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));`,意味着这两种类型将不会做缓存。

```java
public RequestBuilder<TranscodeType> load(@Nullable Bitmap bitmap) {
  return loadGeneric(bitmap)
      .apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
}
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model;
  isModelSet = true;
  return this;
}
```

总结一下load作用，构造一个RequestBuilder实例，同时传入需要加载的数据源类型。

### 2.3 into
```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread();
  Preconditions.checkNotNull(view);
  BaseRequestOptions<?> requestOptions = this;
  //检查是否额外设置了imageview的裁剪方法
  if (!requestOptions.isTransformationSet()
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    switch (view.getScaleType()) {
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }
  return into(
      //构建target
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions,
      Executors.mainThreadExecutor());
}

@NonNull
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
  return into(target, /*targetListener=*/ null, Executors.mainThreadExecutor());
}
@NonNull
@Synthetic
<Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    Executor callbackExecutor) {
  return into(target, targetListener, /*options=*/ this, callbackExecutor);
}
```
into方法同样是在RequestBuilder内，是glide加载图片流程的最后一步，他暴露了两种public方法，一个的参数是imageview，作用是指定图片最后要加载到的位置。另一个参数是target对象，可以定制化一个target并返回。

对于参数imageview的into方法，首先先检查requestBuilder是否额外设置过imageview的scaletype属性，如果有则在requestoption里面加上裁剪选项，接着构建一个target实例并创建一个主线程的executor用于获得图片资源在主线程更新UI，调用private的into方法。

创建target方法细节如下：
```java
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
    @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
        //工厂方法创建target的子类viewtarget
  return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}
public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
    @NonNull Class<Z> clazz) {
//根据传入的不同class类型构造bitmap或者drawabletarget
  if (Bitmap.class.equals(clazz)) {
    return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
  } else if (Drawable.class.isAssignableFrom(clazz)) {
    return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
  } else {
    throw new IllegalArgumentException(
        "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
  }
}

public class DrawableImageViewTarget extends ImageViewTarget<Drawable> {
  public DrawableImageViewTarget(ImageView view) {
    super(view);
  }
  /**
   * @deprecated Use {@link #waitForLayout()} instead.
   */
  // Public API.
  @SuppressWarnings({"unused", "deprecation"})
  @Deprecated
  public DrawableImageViewTarget(ImageView view, boolean waitForLayout) {
    super(view, waitForLayout);
  }
  //负责将最后获得Drawable资源加载到into指定的imageview上
  @Override
  protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
  }
}
```

对于参数target的into方法，则是直接创建一个主线程的runnable用于回调target给主线程。

接下来看private的into方法做了什么：
```java
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    BaseRequestOptions<?> options,
    Executor callbackExecutor) {
  Preconditions.checkNotNull(target);
  //判断是否调用过load方法设置目资源
  if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }
  //构建Request实例
  Request request = buildRequest(target, targetListener, options, callbackExecutor);
  Request previous = target.getRequest();
  if (request.isEquivalentTo(previous)
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();
    if (!Preconditions.checkNotNull(previous).isRunning()) {
    //使用上一个请求而不是新请求来允许优化，例如跳过设置占位符，跟踪和取消跟踪目标，以及获取在单个请求中完成的查看维度。
      previous.begin();
    }
    return target;
  }
  //取消挂起的任务 清除资源 达到复用目的
  requestManager.clear(target);
  //为target绑定request
  target.setRequest(request);
  //执行request
  requestManager.track(target, request);
  return target;
}
```
先判断是否调用过load方法设置目标资源变量，如果没有直接抛出异常，接着构建request实例，同时获得target上的前一个request（参数为imageview的into方法跳过），如果相同则直接复用前一个request，免去了一些配置步骤，同时为了能顺利完成回调，增加了重试机制。然后对于imageview来说会先取消之前挂起的任务清楚任务资源，然后为target重新绑定request请求，track方法开始执行request任务。

看一下最后track方法做了什么：
```java
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
  //将target加入到追踪队列
  targetTracker.track(target);
  //执行request请求
  requestTracker.runRequest(request);
}
```

首先会将目标target加入到追踪队列，这个队列里保存了当前activity里所有的target，同时和生命周期进行了绑定，这样做的好处是用生命周期自动管理了request请求的开始、暂停、结束等操作。
```java
public final class TargetTracker implements LifecycleListener {
  private final Set<Target<?>> targets =
      Collections.newSetFromMap(new WeakHashMap<Target<?>, Boolean>());
  public void track(@NonNull Target<?> target) {
    targets.add(target);
  }
  public void untrack(@NonNull Target<?> target) {
    targets.remove(target);
  }
  @Override
  public void onStart() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStart();
    }
  }
  @Override
  public void onStop() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onStop();
    }
  }
  @Override
  public void onDestroy() {
    for (Target<?> target : Util.getSnapshot(targets)) {
      target.onDestroy();
    }
  }
  @NonNull
  public List<Target<?>> getAll() {
    return Util.getSnapshot(targets);
  }
  public void clear() {
    targets.clear();
  }
}
```

runRequest方法则是最终真正执行加载图片资源的操作：
```java
public void runRequest(@NonNull Request request) {
  requests.add(request);
  //request队列是否处于暂停状态
  if (!isPaused) {
    //如果不则执行request
    request.begin();
  } else {
    //清楚request资源
    request.clear();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Paused, delaying request");
    }
    //加入挂起队列
    pendingRequests.add(request);
  }
}
```

begin操作是调用了SingleRequest类的begin方法，SingleRequest实现了Request接口：
```java
public synchronized void begin() {
  assertNotCallingCallbacks();
  stateVerifier.throwIfRecycled();
  startTime = LogTime.getLogTime();
  //如果
  if (model == null) {
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      width = overrideWidth;
      height = overrideHeight;
    }
    int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
    onLoadFailed(new GlideException("Received null model"), logLevel);
    return;
  }
  //正在运行 重复执行Request抛出异常
  if (status == Status.RUNNING) {
    throw new IllegalArgumentException("Cannot restart a running request");
  }
  //如果我们在完成之后重新启动（通常通过诸如notifyDataSetChanged之类的东西，在相同的目标或视图中启动相同的请求），我们可以简单地使用我们上次检索的资源和大小，并跳过获取新的大小
  //这意味着想要重新启动负载因为期望视图大小已更改的用户需要在开始新加载之前明确清除视图或目标。
  if (status == Status.COMPLETE) {
    onResourceReady(resource, DataSource.MEMORY_CACHE);
    return;
  }
  // Restarts for requests that are neither complete nor running can be treated as new requests
  // and can run again from the beginning.
  status = Status.WAITING_FOR_SIZE;
  if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
    onSizeReady(overrideWidth, overrideHeight);
  } else {
    target.getSize(this);
  }
  if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
      && canNotifyStatusChanged()) {
    target.onLoadStarted(getPlaceholderDrawable());
  }
}
```
方法里首先判断是否设置过model，如果没有则直接回调加载失败，然后判断是否正在运行，如果重复请求就直接抛出异常，接着判断是否Request已经完成，完成则调用`onResourceReady`,接着给view确定height和width，同时调用`onSizeReady`，如果状态处于running或者WAITING_FOR_SIZE，调用`onLoadStarted`。

这三个回调的名字很明显，分别对应Request的三个过程，onSizeReady（准备）、onLoadStarted（开始）、onResourceReady（资源完成），一个个来看。

#### 2.3.1 onSizeReady
```java
public synchronized void onSizeReady(int width, int height) {
  stateVerifier.throwIfRecycled();
  //如果进入时不是WAITING_FOR_SIZE直接退出
  if (status != Status.WAITING_FOR_SIZE) {
    return;
  }
  //状态调整至running
  status = Status.RUNNING;
  float sizeMultiplier = requestOptions.getSizeMultiplier();
  this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
  this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
  loadStatus =
      engine.load(
          glideContext,
          model,
          requestOptions.getSignature(),
          this.width,
          this.height,
          requestOptions.getResourceClass(),
          transcodeClass,
          priority,
          requestOptions.getDiskCacheStrategy(),
          requestOptions.getTransformations(),
          requestOptions.isTransformationRequired(),
          requestOptions.isScaleOnlyOrNoTransform(),
          requestOptions.getOptions(),
          requestOptions.isMemoryCacheable(),
          requestOptions.getUseUnlimitedSourceGeneratorsPool(),
          requestOptions.getUseAnimationPool(),
          requestOptions.getOnlyRetrieveFromCache(),
          this,
          callbackExecutor);
  if (status != Status.RUNNING) {
    loadStatus = null;
  }
}
```
load方法就是真正执行加载资源的代码,里面有一个runWrapped方法：
```java
private void runWrapped() {
  switch (runReason) {
    case INITIALIZE:
    //获取下一个状态
      stage = getNextStage(Stage.INITIALIZE);
      currentGenerator = getNextGenerator();
      runGenerators();
      break;
    case SWITCH_TO_SOURCE_SERVICE:
      runGenerators();
      break;
    case DECODE_DATA:
      decodeFromRetrievedData();
      break;
    default:
      throw new IllegalStateException("Unrecognized run reason: " + runReason);
  }
}
private enum Stage {
  /** The initial stage. */
  INITIALIZE,
  /** Decode from a cached resource. */
  RESOURCE_CACHE,
  /** Decode from cached source data. */
  DATA_CACHE,
  /** Decode from retrieved source. */
  SOURCE,
  /** Encoding transformed resources after a successful load. */
  ENCODE,
  /** No more viable stages. */
  FINISHED,
}
```
这里做了一个状态机的转换，按照Stage的流程不断的流转，如果runReason是INITIALIZE，就获取Stage.INITIALIZE的下一个状态，先从RESOURCE_CACHE内存里获取缓存，再从DATA_CACHE磁盘获取缓存，再从source数据源取数据。这就是三级缓存的策略。

三级缓存的生成对应着三个生成类，通过调用getNextGenerator方法获取：
```java
private DataFetcherGenerator getNextGenerator() {
  switch (stage) {
    case RESOURCE_CACHE:
      return new ResourceCacheGenerator(decodeHelper, this);
    case DATA_CACHE:
      return new DataCacheGenerator(decodeHelper, this);
    case SOURCE:
      return new SourceGenerator(decodeHelper, this);
    case FINISHED:
      return null;
    default:
      throw new IllegalStateException("Unrecognized stage: " + stage);
  }
}
```
获取到DataFetcherGenerator之后，就会调用runGenerators方法去执行获取数据操作,startNext方法就是内部获取数据代码。
```java
private void runGenerators() {
  currentThread = Thread.currentThread();
  startFetchTime = LogTime.getLogTime();
  boolean isStarted = false;
  while (!isCancelled && currentGenerator != null
      && !(isStarted = currentGenerator.startNext())) {
    stage = getNextStage(stage);
    currentGenerator = getNextGenerator();
    if (stage == Stage.SOURCE) {
      reschedule();
      return;
    }
  }
  // We've run out of stages and generators, give up.
  if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
    notifyFailed();
  }
}
```

来看一下SourceGenerator的startNext()，会调用loadData方法，根据不同的获取资源策略加载数据，在HttpUrlFetcher类里也就是网络请求数据的loadData方法中，会请求url拿到输入流，然后回调给Generator，Generator的onDataReady方法接收到回调之后会根据缓存策略选择将数据缓存起来或是回调数据给外部。
```java
public boolean startNext() {
    //更新缓存
  if (dataToCache != null) {
    Object data = dataToCache;
    dataToCache = null;
    cacheData(data);
  }
  if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
    return true;
  }
  sourceCacheGenerator = null;
  loadData = null;
  boolean started = false;
  while (!started && hasNextModelLoader()) {
    loadData = helper.getLoadData().get(loadDataListIndex++);
    if (loadData != null
        && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
        || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
      started = true;
      //加载数据
      loadData.fetcher.loadData(helper.getPriority(), this);
    }
  }
  return started;
}

//HttpUrlFetcher.java
public void loadData(@NonNull Priority priority,
    @NonNull DataCallback<? super InputStream> callback) {
  long startTime = LogTime.getLogTime();
  try {
    //获得输入流
    InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
    //回调
    callback.onDataReady(result);
  } catch (IOException e) {
    if (Log.isLoggable(TAG, Log.DEBUG)) {
      Log.d(TAG, "Failed to load data for url", e);
    }
    callback.onLoadFailed(e);
  } finally {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
    }
  }
}

public void onDataReady(Object data) {
  DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
  //缓存策略是可以缓存数据的话就缓存数据
  if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
    dataToCache = data;
    // We might be being called back on someone else's thread. Before doing anything, we should
    // reschedule to get back onto Glide's thread.
    cb.reschedule();
  } else {
    //不能缓存就仍旧向外回调数据
    cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
        loadData.fetcher.getDataSource(), originalKey);
  }
}
```
#### 2.3.2 onLoadStarted
当status的状态为running或者WAITING_FOR_SIZE的时候，就会调用该方法，它会调用target的onLoadStarted做一些准备工作，在`ImageViewTarget`类中就会设置placeholder和一些加载动画。
```java
public void onLoadStarted(@Nullable Drawable placeholder) {
    super.onLoadStarted(placeholder);
    setResourceInternal(null);
    setDrawable(placeholder);
}
```

#### 2.3.3 onResourceReady
这个方法就是最后将获得数据装进imageview或者返回给target的方法：
```java
private synchronized void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
  // We must call isFirstReadyResource before setting status.
  boolean isFirstResource = isFirstReadyResource();
  status = Status.COMPLETE;
  this.resource = resource;
  if (glideContext.getLogLevel() <= Log.DEBUG) {
    Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
        + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
        + LogTime.getElapsedMillis(startTime) + " ms");
  }
  isCallingCallbacks = true;
  try {
    boolean anyListenerHandledUpdatingTarget = false;
    if (requestListeners != null) {
      for (RequestListener<R> listener : requestListeners) {
        anyListenerHandledUpdatingTarget |=
            listener.onResourceReady(result, model, target, dataSource, isFirstResource);
      }
    }
    anyListenerHandledUpdatingTarget |=
        targetListener != null
            && targetListener.onResourceReady(result, model, target, dataSource, isFirstResource);
    if (!anyListenerHandledUpdatingTarget) {
      Transition<? super R> animation =
          animationFactory.build(dataSource, isFirstResource);
    // 1
      target.onResourceReady(result, animation);
    }
  } finally {
    isCallingCallbacks = false;
  }
  notifyLoadSuccess();
}
```
重点看一下注释1出的target.onResourceReady，它将获取到的数据通过target调用塞入target中进行加载，看一下ImageViewTarget的方法：
```java
public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
    //没有变化 则直接setresource
  if (transition == null || !transition.transition(resource, this)) {
    setResourceInternal(resource);
  } else {
    //有变换 需要更新动画
    maybeUpdateAnimatable(resource);
  }
}
private void setResourceInternal(@Nullable Z resource) {
  // Order matters here. Set the resource first to make sure that the Drawable has a valid and
  // non-null Callback before starting it.
  setResource(resource);
  maybeUpdateAnimatable(resource);
}

protected void setResource(@Nullable Drawable resource) {
  view.setImageDrawable(resource);
}
```

## 3 总结
上文的解析，一套完整的流程就走下来了，但是glide强大之处远不止于此，还有具体的缓存策略没有讲解，后续会继续对glide进行分析。

## 4 参考资料
[Android图片加载框架最全解析（二），从源码的角度理解Glide的执行流程](https://blog.csdn.net/guolin_blog/article/details/53939176)

[Glide github](https://github.com/bumptech/glide)



