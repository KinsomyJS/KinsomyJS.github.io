---
layout:     post                    # 使用的布局（不需要改）
title:   Kotlin(一)Suspend Function          # 标题 
subtitle:   Suspend Function 和 SuspendCancellableCoroutine #副标题
date:       2020-09-01            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - kotlin
---

## 1 什么是Suspend Function？
Suspend function是用`suspend`关键字修饰的函数，suspend function需要在协程中执行，或者在另一个suspend function内。

我们考虑下面这样的场景，用户先输入账号密码进行注册，然后再用账号密码替用户自动登录，接着账户信息保存到本地缓存，最后返回用户信息。

如果按照以往用RxJava的写法，我们会写出如下的代码：
```kotlin
    fun newUser(userName: String, password: String, callback: Callback<User>) {
        // 注册请求
        httpClient.register { user ->
            // 登录请求
            httpClient.login(user) { user ->
                // 保存结果到本地
                dataBase.save(user)
                // 回调
                userResult.success(user)
            }
        }
    }
```
这样的rxjava链式调用，我们都再熟悉不过了，一层层的回调向下传递。

如果用了suspend function，我们就可以把它看成一个普通的方法，只是这个方法可以被挂起和在任务完成后恢复，这意味着我们可以将一个耗时任务放到suspend function中等它完成。这样带来的好处是不用再像上面那样写回调嵌套，而是可以顺序调用每一个异步方法，可以写出下面这样的代码：

```kotlin
    suspend fun newUser(name: String, pwd: String): User {
        val registerUser = register(name, pwd)
        val user = login(registerUser)
        dataBase.save(user)
        return user
    }

    suspend fun register(name: String, pwd: String): User

    suspend fun login(user: User): User
```

需要注意的是只需要在耗时方法上增加`suspend`标记，这样可以让阻塞操作变成非阻塞操作，如果只是一个普通方法调用，就会收到一个警告`Redundant 'suspend' modifier `,意思是这个方法可以不用变成suspend function。
```kotlin
    // Redundant 'suspend' modifier
    suspend fun redundant() {
        print("redundant suspend")
    }

    suspend fun redundant() {
        withContext(Dispatchers.Default) {
            print("suspend")
        }
    }
```

## 2 suspendCancellableCoroutine
在kotlin之前，异步请求往往会采用回调函数，将异步线程的数据回调回主线程。

在kotlin中则可以使用`suspendCancellableCoroutine`将回调函数转换成协程，`suspendCancellableCoroutine`方法返回了`CancellableContinuation`实例。

```kotlin
public interface CancellableContinuation<in T> : Continuation<T> {
    public val isActive: Boolean

    public val isCompleted: Boolean

    public val isCancelled: Boolean

    public fun cancel(cause: Throwable? = null): Boolean

    @ExperimentalCoroutinesApi // since 1.2.0
    public fun resume(value: T, onCancellation: ((cause: Throwable) -> Unit)?)
}

@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

/**
 * Resumes the execution of the corresponding coroutine so that the [exception] is re-thrown right after the
 * last suspension point.
 */
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resumeWithException(exception: Throwable): Unit =
    resumeWith(Result.failure(exception))
```

CancellableContinuation有三种状态
| **State**                           | [isActive] | [isCompleted] | [isCancelled] |
| ----------------------------------- | ---------- | ------------- | ------------- |
| _Active_ (initial state)            | `true`     | `false`       | `false`       |
| _Resumed_ (final _completed_ state) | `false`    | `true`        | `false`       |
| _Canceled_ (final _completed_ state)| `false`    | `true`        | `true`        |

下面是三种状态之间的转换关系：
```
   +-----------+   resume    +---------+
   |  Active   | ----------> | Resumed |
   +-----------+             +---------+
         |
         | cancel
         V
   +-----------+
   | Cancelled |
   +-----------+
```
看一个简单的例子，讲解都在注释中，很好理解，用resume和resumeWithException就可以在协程里用回调写法。
```java
MainScope().launch {
  try {
    val user = fetchUser()
    updateUser(user)
  } catch (exception: Exception) {
    // 捕获resumeWithException抛出的异常
  }
}

private suspend fun fetchUser(): User = suspendCancellableCoroutine { 
cancellableContinuation ->
  // 异步网络请求
  fetchUserFromNetwork(object : Callback {
    override fun onSuccess(user: User) {
      // 回调获得的数据，最后会被fetchUser()方法声明的val user变量接收到
      cancellableContinuation.resume(user)
    }

    override fun onFailure(exception: Exception) {
      //网络请求失败，抛出异常会被上面的try/catch块捕获
      cancellableContinuation.resumeWithException(exception)
    }
  })
}

private fun fetchUserFromNetwork(callback: Callback) {
  Thread {
    Thread.sleep(3_000)
    
    //模拟网路响应的回调
    callback.onSuccess(User())
  }.start()
}

private fun updateUser(user: User) {
  // 更新ui
}

interface Callback {
  fun onSuccess(user: User)
  fun onFailure(exception: Exception)
}

class User
```

`suspendCancellableCoroutine`比`suspendCoroutine`多了一个`cancel`方法,可以手动取消掉Continuation的执行，让流程变得更加可控。`cancel`会抛出一个`CancellationException`异常，这个异常不会导致crash，但是会让协程取消，后续代码都不会执行，如果用try/catch捕获异常，则后续代码可以继续执行。

## 3 参考资料
* [Kotlin Coroutines in Android — Suspending Functions](https://medium.com/swlh/kotlin-coroutines-in-android-suspending-functions-8a2f980811f8)
