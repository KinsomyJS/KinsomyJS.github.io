---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(四)LiveData解析          # 标题 
subtitle:   Android advance #副标题
date:       2019-02-25            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 概述
LiveData是一个可被观察的数据持有类，一般的数据类不同，LiveData是生命周期感知的，数据类的生命周期可以和其他app组件的生命周期保持一致，例如Activity，fragment和service。这保证了LiveData仅仅会更新处在活动状态的组件。

LiveData可以被看成观察者模式的实践，LiveData是一个被观察的对象，其他组件会订阅对它的观察，当组件处于`Started`或者`Resumed`则为活跃状态，LiveData只会通知处于这两者状态的组件更新。同时希望在组件变为`Destroyed`状态时去自动销毁LiveData和组件的绑定，不至于出现泄漏。

### 1.1 优势
* 确保UI和数据状态同步
* 没有内存泄漏
* 不会因为已经停止的Activity导致crash
* 不需要手动处理生命周期
* 组件时刻保持最新数据
* 支持适当配置更改

    如果一个Activity或者fragment因为例如设备旋转而重新创建，它会立即接受到最新可获得的数据。

* 共享资源 多个组件可以共享同一份数据

## 2 实践
### 2.1 依赖
LiveData是Android官方架构Jetpack的组成部分，架构组件全部在google的Maven仓库里，想要使用可以在项目的build.gradle文件里添加`google()`依赖。
```gradle
allprojects {
    repositories {
        google() //引入livedata
        jcenter()
    }
}
```
在模块的build.gradle文件中加入lifecycle:extensions依赖
```gradle
implementation "android.arch.lifecycle:extensions:1.1.1"
```

### 2.2 创建LiveData对象
LiveData通常和Jetpack架构下的另一个组件`ViewModel`配合使用，ViewModel是一个负责为Activity或者Fragment准备和管理数据的类，同时处理和应用剩余部分的通信，注意ViewModel仅仅负责管理UI上的数据，其他都无权干涉，它和组件生命周期绑定，只有Activity结束了，它才会被销毁。

我们创建一个提供电池电量信息的ViewModel：
```java
public class BatteryViewModel extends ViewModel {
	private MutableLiveData<Integer> currentBattery;

	public MutableLiveData<Integer> getCurrentBatteryData() {
		if (currentBattery == null) {
			currentBattery = new MutableLiveData<>();
		}

		return currentBattery;
	}
}
```
这里使用了`MutableLiveData`类来保存电量数据，它继承了LiveData，暴露了`setValue`、`postValue`方法。
```java
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

### 2.3 监听LiveData对象
写一个activity模拟显示电量变化情况，通常我们要在onCreate回调里开始监听LiveData，主要出于以下原因：
* onCreate是activity创建时的第一个回调，onResume和onStart在activity生命周期内会回调多次，造成调用监听多次形成冗余。
* 确保activity和fragment在变成活跃状态进入started时可以尽快获得数据更新，所以要尽早开始监听。

LiveData在用非活跃状态进入活跃状态时同样可以接受到更新，但是如果第二次从非活跃状态进入活跃状态，那么只有当上一次编程活跃态的数据发生变化时才会接受更新。

```java
public class BatteryActivity extends AppCompatActivity {
    private int battery = 100;
	private BatteryViewModel mBatteryViewModel;
	@Override
	protected void onCreate(@Nullable Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_livedata_battery);

		//在当前activity范围内创建或者获得viewmodel实例
		mBatteryViewModel = ViewModelProviders.of(this).get(BatteryViewModel.class);
		final TextView tvBattery = findViewById(R.id.tv_battery);
		//设置观察者
		Observer<Integer> batteryOb = new Observer<Integer>() {
			@Override
			public void onChanged(@Nullable Integer integer) {
				tvBattery.setText(String.format("电量：^[0-9]*$", integer));
			}
		};
		//将数据绑定到观察者
		mBatteryViewModel.getCurrentBatteryData().observe(this, batteryOb);
	}
}
```
`ViewModelProviders`是lifecycle:extensions模块下的类，首先通过`of(this)`创建一个ViewModelProvider实例，再通过get方法去获得(如果有)或者创建一个viewmodel，然后创建一个`Observer`，将它和viewmodel绑定，监听数据的变化，在`onChanged`回调内实时修改UI。

### 2.4 更新LiveData数据
监听设置好之后，下面可以通过修改数据看看ui是否被实时修改，点击开始统计按钮，让电量减一来模拟掉电情况。

LiveData更改数据的方法不是public类型的，只在内部自己调用，所有这里才会使用MutableLiveData，他暴露了修改数据的公共方法。这里修改电量减一就是用到了setValue方法。

```java
findViewById(R.id.btn_start).setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(View v) {
				mBatteryViewModel.getCurrentBatteryData().postValue(battery--);
			}
		});
```

可以看到`MutableLiveData`总共有两个修改数据的方法，他们的区别是什么呢？

* setValue是必须在主线程被调用，用来修改LiveData数据。
* postValue可以在后台线程调用，它是向线程的观察者发送一个task，请求修改数据，但是如果在主线程执行前调用多次，则只有最后一次会生效。

## 3 扩展LiveData
上面是使用自带的MutableLiveData，同样也可以自己定义LiveData类。

### 3.1 自定义LiveData
自定义一个BatteryLiveData类继承自LiveData，通过广播去接受系统电池电量，通过setValue将数据设置给LiveData。

```java
public class BatteryLiveData extends LiveData<Integer> {
	private Context context;

	private BroadcastReceiver receiver = new BroadcastReceiver() {
		@Override
		public void onReceive(Context context, Intent intent) {
			setValue(intent.getIntExtra("level", 0));
		}
	};

	public BatteryLiveData(Context context) {
		this.context = context;
	}

	@Override
	protected void onActive() {
		super.onActive();
		IntentFilter filter = new IntentFilter();
		filter.addAction(Intent.ACTION_BATTERY_CHANGED);
		context.registerReceiver(receiver, filter);
	}

	@Override
	protected void onInactive() {
		super.onInactive();
		context.unregisterReceiver(receiver);
	}
}
```

需要重写两个方法：
* onActive() 
    当LiveData绑定有活跃状态的observer时就会调用，在这里回去注册广播获得电池电量变化。
* onInactive()
    当LiveData没有任何活跃状态observer绑定时调用，取消注册广播。

### 3.2 获得数据更新
在activity里构造LiveData实例，同时调用`observe()`方法将activity和LiveData绑定。
```java
BatteryLiveData batteryLiveData = new BatteryLiveData(this);
		batteryLiveData.observe(this, new Observer<Integer>() {
			@Override
			public void onChanged(@Nullable Integer integer) {
				tvBattery.setText(integer.toString());
			}
		});
```

### 3.3 共享资源
文章一开始讲到LiveData的优势之一就是共享资源，可以将LiveData设计成单例模式，在任何需要的地方调用observe()绑定监听。
```java
public class BatteryLiveData extends LiveData<Integer> {
    ...
	@MainThread
	public static BatteryLiveData getInstance(Context context) {
		if (sInstance == null) {
			sInstance = new BatteryLiveData(context);
		}
		return sInstance;
	}

	private BatteryLiveData(Context context) {
		this.context = context;
	}
    ...
}


BatteryLiveData.getInstance(this).observe(this, new Observer<Integer>() {
			@Override
			public void onChanged(@Nullable Integer integer) {
				tvBattery.setText(integer.toString());
			}
		});
```

## 4 LiveData转换
### 4.1 map
map是将一个LiveData转换成另一个LiveData。
```java
Transformations.map(batteryLiveData, new Function<Integer, String>() {
			@Override
			public String apply(Integer input) {
				return input + "%";
			}
		});
```

### 4.2 switchMap
第二个参数接收一个方法，通过方法将传入的第一个参数LiveData转换成另一个LiveData。
```java
		Transformations.switchMap(sInstance, new Function<Integer, LiveData<? extends String>>() {
			@Override
			public LiveData<? extends String> apply(Integer input) {
				return ...;
			}
		});
```

## 5 源码解析
下面来分析一下LiveData的关键源码。先从LiveData类开始：
LiveData里面主要有几个上面用到的方法，`observe`,`setValue`,`postValue`,`onActive`,`onInactive`等，挨个来分析。

### 5.1 observe()
```java
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }

public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```
observe有两个参数，第一个是`LifecycleOwner`,[Fragment](https://developer.android.com/reference/android/support/v4/app/Fragment)和[FragmentActivity](https://developer.android.com/reference/android/support/v4/app/FragmentActivity.html)等都实现了该接口，因此我们在调用的时候就可以直接传入this，它里面有一个方法可以获得当前的Lifecycle实例，`Lifecycle`里面保存了和生命周期相对应的状态。

observe方法首先先判断当前状态是不是`DESTROYED`,如果是就可以完全忽略，因为已经说过只对处于活跃状态的组件做更新；接着将owner observer构造成`LifecycleBoundObserver`实例，这是一个内部类，里面有关于状态变换的一系列操作，待会详细分析；然后将observer和wrapper存入map缓存中，如果observer缓存已存在并且已经和另一个`LifecycleOwner`绑定，则抛出异常；如果缓存已经存在则直接忽略；最后调用addObserver方法将`LifecycleBoundObserver`实例和`LifecycleOwner`绑定。而addObserver是调用了`LifecycleRegistry`类的实现。

### 5.2 ObserverWrapper
```java
private abstract class ObserverWrapper {
        final Observer<T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<T> observer) {
            mObserver = observer;
        }

        abstract boolean shouldBeActive();

        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }

        void detachObserver() {
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
			//过去是inactive，现在是active
            if (wasInactive && mActive) {
                onActive();
            }
			//过去没有订阅，并且现在是inactive
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
			//现在是active
            if (mActive) {
                dispatchingValue(this);
            }
        }
    }
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
	...
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }
}
```
ObserverWrapper里面封装了关于状态的操作，包括判断是否处于活跃状态、observer是否绑定到lifecycleowner以及更改activity状态等。

activeStateChanged首先判断新来的状态和旧状态是否相同，相同则忽略，然后判断LiveData上的活跃态的数量是否为0，为0说明之前处于Inactive，然后统计现在的订阅数，接着就是三个if判断，注释在代码里。正式这三个判断，LiveData可以接收到onActive和onInactive的回调。

dispatchingValue(this)是当状态变为active时调用，用来更新数据。里面会用到considerNotify方法。

### 5.3 setValue postValue
```java
private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}


@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```

setValue会调用dispatchingValue方法，接着调用considerNotify，在最后调用onChange()回调，就能收到数据变化。

postValue之前说过是往主线程发送事件，同时加锁保持占用，防止多线程并发竞争导致的数据错误，因为每次postValue成功都会对mPendingData重置为NOT_SET。然后想主线程发送Runnable对象，Runnable实例的run方法会执行setValue在主线程修改数据。



## 6 参考资料
* [LiveData Overview](https://developer.android.com/topic/libraries/architecture/livedata#java)

* [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html)


