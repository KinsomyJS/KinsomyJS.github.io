---
layout:     post                    # 使用的布局（不需要改）
title:      Flutter 探索与实践               # 标题 
subtitle:   Hello Flutter #副标题
date:       2018-09-25              # 时间
author:     Kinsomy                      # 作者
header-img: img/flutter-bg.jpeg   #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - flutter
    - dart
    - android
    
---

Flutter是google近年来新推出的跨平台移动UI框架，可以在ios和Android系统上快速构建出高质量，体验较高的**原生界面**，同时Flutter还将会作为google新一代操作系统Fuchsia的Toolchain，这对Flutter的未来发展前景是一个强有力的支撑。写这篇文章时，中国 GDG 2018 刚刚落幕，Flutter团队在大会上发布了release之前的最后一个preview版本。扇贝移动团队近期也对Flutter如何应用到扇贝产品上做了一番探索和实践。

## Hello Flutter
Flutter项目可以用同一套代码同时在ios和Android操作系统上运行，在此之前，市面上已经存在了很多跨平台解决方案，扇贝的移动产品上在跨平台开发上一直采用Webview加React Native的解决方案。
  
Webview的优缺点都非常明显，优势是可以完全用Web开发的技术栈去实现，app只需要提供webview组件承载即可，同时可以随时更新页面，动态性很高，并且这些页面可以完全交由web开发人员开发迭代，移动端不需要太关心，但是Webview的缺点又很多，渲染的性能一直被开发者诟病，内置的Chrome内核在国内的机型上也会有不兼容的情况发生。

React Native在活动页面和商品页面得到了使用，RN相对于Webview来说，性能和渲染效率都得到了不小的提升，它是将编写好的web界面利用JSCore转换成原生控件，同时配合动态更新系统可以发布新版本的RN包来实现动态更新界面，但是RN和系统的关联性较强，很多功能需要依赖系统特性开发，debug成本也相对比较高，移动端的同学也需要对RN有一些了解来配合前端开发。

因此我们将目光投向了Flutter。
### Flutter的特点
* Dart可以运行前编译（AOT），在开发flutter应用的时候布局文件会直接通过源码编写node tree，从而避免了大量的解析转译时间，使得Dart的效率比JS更高。
* Dart语言同样支持JIT编译，因此flutter可以hot reload，为开发周期提速。
* Dart没有锁的概念，可以做到对象回收和GC，Dart中的线程叫做isolates，因为不共享内存的原因，同时和js一样是单线程操作，所以不会出现抢占调度和锁死的问题。开发者控制线程的时候需要显式创建线程，最常用的是async和await。
* Flutter用Dart语言开发，因为Flutter主要用来开发用户界面，Dart语言的特性适合了用户构建用户界面时的操作逻辑，没有像Android的xml文件和前端的html文件这样的单独布局文件，使得开发更简洁，预览更方便。
* Flutter不再受限于native，自己开发了一套渲染逻辑，因此在未来的性能优化和跨平台相比RN优势会更加明显。

关于Flutter的具体使用可以前往官网学习 [https://flutter.io](https://flutter.io)

Flutter的依赖是在Pub仓库中管理，项目的依赖在根目录下的**pubspec.yaml**中进行配置，例如如下配置:
```
dependencies:
  flutter:
    sdk: flutter
  json_annotation: ^1.0.0
  intl: ^0.15.7
```
关于依赖的package版本可以有两种写法
* packagename: ^version 引入某个版本的package
* packagename: ‘>=version’ 引入某个版本之后的package，用来约束最低或最高版本

在pubspec.yaml文件中声明依赖之后需要执行package get命令，国内开发者可能需要科学上网才能拉下来依赖包，Flutter为国内开发者提供了本地化的网站和镜像。只需要简单配置即可。
[using Flutter In China](https://github.com/flutter/flutter/wiki/Using-Flutter-in-China)


## 技术预研
在对Flutter方案做技术预研的时候，我们罗列出了一些需要探索和解决的问题。
* 重写界面的成本
* Flutter和原生互相通信问题
* 图片资源增加包体积
* Flutter如何构建到现有项目

### 重写界面成本
既然开始选用Flutter作为预备跨平台方案，而且是Android和ios团队负责开发维护，就需要了解一下Flutter的学习和开发成本。

Dart是Flutter前期在对十几种语言进行调研之后最后选择的语言，它天然支持响应式编程，Dart2的升级是进一步优化客户端的开发，编写出的代码结构清晰，有一定程序设计基础的开发人员不需要经过系统的学习就可以上手进行Flutter开发。

Flutter提供了大量基于Android material design风格和ios Cupertino风格的控件，可以通过组合嵌套的方式构建界面，需要注意的是在Flutter里面属性也是控件的概念，例如Padding、Center、Align等。具体查看[Widgets Catalog](https://flutter.io/widgets)。

### Flutter和原生互相通信
以Android为例，Dart调用Java代码可以通过MethodChannel来实现，Java调用dart则用EventChannel来实现。

因为需要用Flutter重写的页面有涉及到网络请求的问题，之前的Android项目里实现了一套完整的网络请求框架，封装了项目需要的一些附带请求信息和处理逻辑等，我们目前还不需要为Flutter项目单独实现一个完整网络请求框架，所以将网络请求通过dart调用java的方式在java层进行网络请求然后callback到flutter页面上，有因为网络请求是异步的，dart的async/await关键字可以提供异步支持。

#### MethodChannel
首先在dart代码中实例化MethodChannel对象，然后在需要调用java代码的时候调用invokeMethod方法。
```dart
static const methodChannel = const MethodChannel('com.shanbay.shared.data/method'); Future<ProfileModel> _getProfileJson() async {
    mProfileJson = await platform.invokeMethod("getProfileJson");
    if (mProfileJson != null) {
      setState(() {
        Map profileMap = json.decode(mProfileJson);
        mProfileData = new ProfileModel.fromJson(profileMap);
      });
    }
  }
```
在java中注册MethodChannel，注意channel的名字需要相同。
```java
private static final String METHOD_CHANNEL = "com.shanbay.shared.data/method";
private MethodChannel mMethodChannel;
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	GeneratedPluginRegistrant.registerWith(this);
	mMethodChannel = new MethodChannel(getFlutterView(), METHOD_CHANNEL);
	......
}
mMethodChannel.setMethodCallHandler(
    MethodChannel.MethodCallHandler() {
	@Override
	public void onMethodCall(MethodCall call, final MethodChannel.Result result) {
	    if(call.method.equals("getProfileJson"){
	        //do something
	        result.success(json);
	    }
}
```

#### EventChannel
有时候在页面接收到了一个event事件要动态刷新或者修改页面，需要java调用dart。
在dart代码中实例化EventChannel对象
```
static const eventChannel = const EventChannel('com.shanbay.shared.data/event');
StreamSubscription _subscription = null;
  @override
  void initState() {
    super.initState();
    //开启监听
    if (_subscription == null) {
      _subscription = eventChannel
          .receiveBroadcastStream()
          .listen(_onEvent, onError: _onError);
    }
  }
  
  @override
  void dispose() {
    super.dispose();
    //取消监听
    if (_subscription != null) {
      _subscription.cancel();
    }
  }

  void _onEvent(Object event) {
    setState(() {
      if (event.toString() == "refresh") {
        _getProfileJson();
      }
    });
  }

  void _onError(Object error) {
    setState(() {
      print(error);
    });
  }

```
在java代码中实例化相同一个EventChannel对象获得event实例用来调用dart。
```java
private static final String EVENT_CHANNEL = "com.shanbay.shared.data/event";
private EventChannel mChannel;
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	GeneratedPluginRegistrant.registerWith(this);
	mChannel = new EventChannel(getFlutterView(), EVENT_CHANNEL);
	mChannel.setStreamHandler(new EventChannel.StreamHandler() {
		@Override
		public void onListen(Object arguments, EventChannel.EventSink events) {
			mEvent = events;
		}

		@Override
		public void onCancel(Object arguments) {
		}
	});
}

onEvent(event) {
    mEvent.success("refresh");
}
```

### 图片资源增加包体积
在我们现有的Android项目中往往只提供了xxhdpi分辨率的图片资源，这样做是为了减小apk包体积，让其他分辨率的手机通过设备自适应。Flutter对于图片资源需要提供1x、2x、3x三种分辨率格式，这样如果需要大量本地图片资源，会增加一定的包体积。

美团技术发表过一篇文章，他们针对flutter中需要复用的图片资源采取的方案是对移动app assets中的图片按屏幕密度缩放并且存放到本地，然后dart中调用本地图片。

这样的方案在启动Flutter页面之前就需要知道哪些图片需要被存储到本地，还要做一个图片缩放和存储的操作，我认为这样可能不能确保图片的准确性，同时需要Android和ios两端同时支持，写两份代码，并且还要针对不同的flutter页面记录一个资源文件到页面的映射表，成本有点过高，在我们团队内实现有些繁琐，而且我们要实践的页面只用到了少量的本地图片，占用体积也不是很大，针对这种情况，可以直接用官方的三种分辨率，保证了Android、ios的两端的同步。

### Flutter如何构建到现有项目
这个版块我们计划是采用将flutter项目打包成aar到Android项目中集成的方式构建，其中踩到了一些坑，正在梳理，下一篇文章会详细说明。

## 在个人信息页的实践

![](https://user-gold-cdn.xitu.io/2018/9/25/1660fa2b4d37b48f?w=254&h=382&f=png&s=30659)
这个是我们flutter项目结构，host是一个debug工程，可以直接编译运行，lib文件夹则是dart代码。

最后形成的个人信息页效果图如下：

![](https://user-gold-cdn.xitu.io/2018/9/25/1660fa6393eb39fd?w=540&h=960&f=jpeg&s=36402)

## 总结
Flutter的控件目前只提供了md风格的基础组件，想要自定义控件相对于Android和ios来说还是有一些复杂度，但是Flutter团队通过原生渲染界面确实打破了原有的跨平台解决方案的思路，性能和效率的提升是显而易见的，相比RN来说具有很大的优势。

dart目前在Android Studio上还不能支持代码块折叠，同时格式化还要手动点击，没有快捷键，代码块折叠这一点对dart组合嵌套写界面来说实在是太有必要了，希望可以后续不断优化开发体验。

Flutter的引入无疑会增加包体积，preview2的发布，官方宣布release包体积将会再减小30%，一个空的release项目只需要4.7MB 的体积，对于现在流量吃到饱的情况，其实包体积压缩这个话题可以慢慢的弱化了，优化用户体验是最重要的。

Flutter在扇贝听力上的实践已经打出了release包等待发布，我们会逐渐完善Flutter项目，在更多跨平台场景上使用Flutter开发。

## 参考资料
1.[Flutter 文档]("https://flutter.io")

2.[Flutter Packages]("https://pub.dartlang.org/flutter")

3.[Flutter原理与实践]("https://tech.meituan.com/waimai-flutter-practice.html")