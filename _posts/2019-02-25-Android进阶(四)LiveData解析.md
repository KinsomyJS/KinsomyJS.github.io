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

## 5 参考资料
* [LiveData Overview](https://developer.android.com/topic/libraries/architecture/livedata#java)

* [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html)


