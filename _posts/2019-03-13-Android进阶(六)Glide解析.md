---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(六)Glide解析          # 标题 
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
```
这里的区别主要是将glide和不同的上下文的声明周期绑定，如果是Application或者不在主线程调用，那requetmanager的生命周期和Application相关，否则则会和当前页面的fragmentManager的声明周期相关。因为Activity下fragmentManager的生命周期和Activity相同。所以不管是Activity还是fragment，最后都会委托给fragmentManager做生命周期的管理。

总结来说with方法的作用就是获得当前上下文，构造出和上下文生命周期绑定的requestmanager，自动管理glide的加载开始和停止。

### 2.2 load
load方法也是一组重载方法，定义在`interface ModelTypes<T>`接口里，这是一个泛型接口，规定了load想要返回的数据类型，`RequestManager`类实现了该接口，泛型为Drawable类

