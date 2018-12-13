---
layout:     post                    # 使用的布局（不需要改）
title:      TinyDownloader Android下载器             # 标题 
subtitle:   TinyDownloader  #副标题
date:       2018-05-02              # 时间
author:     kinsomy                     # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签   
    - android

---

# Muses
Muses是一个使用方便的Android下载器框架。

[项目地址](https://github.com/KinsomyJS/Muses)

### Muses有以下优点：

* 支持在Activity、Service、Fragment、Dialog、popupWindow、Notification等组件中使用
* 支持HTTP断点续传
* 多任务自动调度管理

### 截图：

![image](http://upload-images.jianshu.io/upload_images/2481737-44ee3d6a39c95d4d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 基本使用：
#### 依赖：
```java
compile 'com.kinsomy:Muses:1.0.0'
```
#### step1：申请权限
由于Muses是一个网络下载框架，所以会涉及到网络请求以及文件读写。所以使用之前要申请以下权限。

**如果你需要适配Android6.0及以上机型，还需要动态申请权限**。

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

#### step2：注册广播监听器

```java
private DownloadReceiver mDownloadReceiver;
mDownloadReceiver = new DownloadReceiver();
mDownloadReceiver.register(this);

//自定义Receiver继承AbsNewDownloadReceiver,接受回调
private class DownloadReceiver extends AbsNewDownloadReceiver {
		@Override
		public void onTaskErrorEvent(NewDownloadTask task, int code) {
		}

		@Override
		public void onTaskCancelEvent(NewDownloadTask task) {
		}

		@Override
		public void onTaskPauseEvent(NewDownloadTask task) {
		}

		@Override
		public void onTaskCompletedEvent(NewDownloadTask task) {
		}

		@Override
		public void onTaskStartEvent(NewDownloadTask task) {
		}

		@Override
		public void onTaskDownloadingEvent(NewDownloadTask task, boolean showProgress) {
		}
	}
```

#### step3：创建下载任务

```java
//首先实例化manager
private DownloadManager mManager;
mManager = new DownloadManager(this);

//调用manager的方法，传入文件夹、文件名、下载链接、id（可为空）
DownloadTask task = mManager.addDownloadTask(dir, fileName, url, id);
```
这样就可以创建一个下载任务了，我的设计思想是，使用者自己创建的task将由使用者自行管理，对于task的运行将交由manager管理。

这样做的好处是可以实现高度的定制化，使用者完全可以根据自己的需要来操作task。


#### step4:开始下载任务

```java
mManager.startTask(task);

```
#### 取消任务

```java
mManager.cancel(taskId);
```

#### 暂停任务

```java
mManager.pause(taskId);
```

#### 恢复任务

```java
mManager.resume(taskId);
```

Version Log
-------
v_1.0.0 : 下载器基本功能实现

License
-------

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
