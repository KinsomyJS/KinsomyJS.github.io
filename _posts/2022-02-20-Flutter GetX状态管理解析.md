---
layout:     post                    # 使用的布局（不需要改）
title:   Flutter GetX状态管理解析          # 标题 
subtitle:   Flutter GetX Source Code #副标题
date:       2022-02-20            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - Flutter
    - Hybrid
---

## 1 什么是状态管理？
要了解状态管理，我们先要了解Flutter里面的状态（State）是什么？从[官方文档](https://docs.flutter.dev/development/data-and-backend/state-mgmt/ephemeral-vs-app)给出定义如下：

    In the broadest possible sense, the state of an app is everything that exists in memory when the app is running. This includes the app’s assets, all the variables that the Flutter framework keeps about the UI, animation state, textures, fonts, and so on. While this broadest possible definition of state is valid, it’s not very useful for architecting an app.

    First, you don’t even manage some state (like textures). The framework handles those for you. So a more useful definition of state is “whatever data you need in order to rebuild your UI at any moment in time”. Second, the state that you do manage yourself can be separated into two conceptual types: ephemeral state and app state.

从广义上来说，应用程序的状态是应用程序运行时内存中存在的所有内容。这包括应用程序的资源、Flutter 框架保持的有关 UI、动画状态、纹理、字体等的所有变量。虽然这种最广泛的状态定义是有效的，但它对于构建应用程序并不是很有用。

但是，你其实不需要管理一些状态（如纹理等），framework层已经帮你处理好这些。所以我们实际用到的状态就是`构建UI所需要的数据`，同时，状态分为两种：临时状态和app级别的状态。

了解了状态的定义，那么状态管理的必要性就很明显了。

状态管理用来解决app里面的状态更新问题，是用来追踪数据的变化，然后将更新的状态重绘到app的组件上以更新UI显示，又或是app有时需要在不同界面中共享应用程序的状态。
状态管理的目标是使组件状态不被全局化，保证组件独立，数据不外溢，让代码可扩展，易维护，以最小的范围重构UI，做到局部优化。

## 2 常见状态管理框架
状态管理的底层实现一般分为如下几种：
* State ：如 StatefulWidget 、 StreamBuilder 状态管理方式；
* InheritedWidget 专门负责 Widget 树中数据共享的功能型 Widget ：如 Provider 、
scoped_model 就是基于它开发；
* Notification ：与 InheritedWidget 正好相反， InheritedWidget 是从上往下传递数据，
Notification 是从下往上，但两者都在自己的 Widget 树中传递，无法跨越树传递；
* Stream 数据流 ：如 Bloc 、 flutter_redux 、 fish_redux 等也都基于它来做实现；

Flutter经过这几年的发展，社区内已经出现了多款比较优秀的状态管理框架，官方推荐的有如下几款：
* Provider
  
    InheritedWidget 的包装器，使它们更易于使用和更可重用。

* Riverpod
    
    Riverpod 是另一个不错的选择，它类似于 Provider，并且是编译安全和可测试的。 Riverpod 不依赖于 Flutter SDK。

* setState 
    
    适用于较小规模 widget 的暂时性状态的基础管理方法。

* Redux

    前端开发人员比较熟悉的状态容器实现

* Fish-Redux

    Fish Redux 是一个基于 Redux 状态管理的组合式 Flutter 应用框架，适用于构建中型和大型应用。

* BLoC / Rx

    基于Stream数据流、观察者模式的实现

* GetIt 

    这是一个用于 Dart 和 Flutter 项目的简单服务定位器，其中包含一些受 Splat 启发的其他好东西。它可以用来代替 InheritedWidget 或 Provider 来访问对象，例如从您的用户界面。

* MobX

    一个基于观察及响应的状态管理常用库。

* Flutter Commands

    基于 ValueNotifiers 的命令式的状态管理，能与 GetIt 完美结合使用，也可以与 Provider 或者其他 locators 配合使用。

* Binder

    一个使用 InheritedWidget 作为核心实现的状态管理库。受到 recoil 的启发，该库提供了分治的解决方式。

* GetX
    
    一个简单的响应式状态管理解决方案。

这里就不一一介绍了，如果感兴趣可以通过[flutter pub仓库](https://pub.flutter-io.cn/)去查找每一个框架的具体文档。

## 3 GetX状态管理解析
我在开发项目之初进行路由管理选型时，高效、简洁，结构明晰、跨页面交互是我比较关心的点，GetX正好满足了我的诉求。

### 3.1 GetX 相关优势 （来自https://xie.infoq.cn/article/0ec29fcd60ac96f15aa0c3180）

* 依赖注入
  * GetX 是通过依赖注入的方式，存储相应的 XxxGetxController；已经脱离了 InheritedWidget 那一套玩法，自己手动去管理这些实例，使用场景被大大拓展
  * 简单的思路，却能产生深远的影响：优雅的跨页面功能便是基于这种设计而实现的、获取实例无需 BuildContext、GetBuilder 自动化的处理及其减少了入参等等
* 跨页面交互
  * 对于复杂的生产环境，跨页面交互的场景，实在太常见了，GetX 的跨页面交互，实现的也较为优雅。
* 路由管理
  * getx 内部实现了路由管理，而且用起来，非常简单
  * GetX 实现了动态路由传参，也就是说直接在命名路由上拼参数，然后能拿到这些拼在路由上的参数，也就是说用 flutter 写 H5，直接能通过 Url 传值
* 实现了全局 BuildContext 
* 国际化，主题实现



Get有两个不同的状态管理器：简单的状态管理器（GetBuilder）和响应式状态管理器（GetX）。
### 3.2 GetBuilder
GetBuilder是手动控制的状态管理器，不使用ChangeNotifier，状态管理器使用较少的内存（接近0mb）。

它针对多状态控制的。想象一下，你在购物车中添加了30个产品，你点击删除一个，同时List更新了，价格更新了，购物车中的徽章也更新为更小的数字。这种类型的方法使GetBuilder成为杀手锏，因为它将状态分组并一次性改变，而无需为此进行任何 "计算逻辑"。GetBuilder就是考虑到这种情况而创建的，因为对于短暂的状态变化，你可以使用setState，而不需要状态管理器。

看一下使用：
```dart
// 创建控制器类并扩展GetxController。
class Controller extends GetxController {
  int counter = 0;
  void increment() {
    counter++;
    update(); // 当调用增量时，使用update()来更新用户界面上的计数器变量。
  }
}
// 在你的Stateless/Stateful类中，当调用increment时，使用GetBuilder来更新Text。
GetBuilder<Controller>(
  init: Controller(), // 首次启动
  builder: (_) => Text(
    '${_.counter}',
  ),
)
//只在第一次时初始化你的控制器。第二次使用ReBuilder时，不要再使用同一控制器。一旦将控制器标记为 "init "的部件部署完毕，你的控制器将自动从内存中移除。你不必担心这个问题，Get会自动做到这一点，只是要确保你不要两次启动同一个控制器。

```
刷新数据的方法在GetXController的update方法，GetBuilder创建的时候接收了一个泛型Controller，内部接收到update()方法后，执行了刷新操作，所以下面深入GetBuilder内部。

**关于源码的解读我都放在了代码的注释里，方便对照代码理解**

```dart
class GetBuilder<T extends GetxController> extends StatefulWidget {
  final GetControllerBuilder<T> builder;
  final bool global;
  final Object? id;
  final String? tag;
  final void Function(GetBuilderState<T> state)? initState,
      dispose,
      didChangeDependencies;
  final void Function(GetBuilder oldWidget, GetBuilderState<T> state)?
      didUpdateWidget;
  final T? init;
  ...

  const GetBuilder({
    Key? key,
    this.init,
    required this.builder,
    this.initState,
    this.tag,
    this.id,
    this.didChangeDependencies,
    this.didUpdateWidget,
    ...
  }) : super(key: key);

  @override
  GetBuilderState<T> createState() => GetBuilderState<T>();
}

class GetBuilderState<T extends GetxController> extends State<GetBuilder<T>>
    with GetStateUpdaterMixin {
  T? controller;
  bool? _isCreator = false;
  VoidCallback? _remove;
  Object? _filter;

  @override
  void initState() {
    /// _GetBuilderState._currentState = this;
    super.initState();
    widget.initState?.call(this);

    /// 判断当前tag的controller是否已经注册过
    var isRegistered = GetInstance().isRegistered<T>(tag: widget.tag);

    if (widget.global) {
      if (isRegistered) {
        if (GetInstance().isPrepared<T>(tag: widget.tag)) {
          _isCreator = true;
        } else {
          _isCreator = false;
        }
        /// 如果已经注册过，直接find出来赋值
        controller = GetInstance().find<T>(tag: widget.tag);
      } else {
        /// 如果没有注册过，调用init参数赋值，init为外部构建GetBuilder返回的Controller()实例
        controller = widget.init;
        _isCreator = true;
        /// 把构建的controller实例加载到Get队列里
        GetInstance().put<T>(controller!, tag: widget.tag);
      }
    } else {
      controller = widget.init;
      _isCreator = true;
      controller?.onStart();
    }

    if (widget.filter != null) {
      _filter = widget.filter!(controller!);
    }

    _subscribeToController();
  }

  /// Register to listen Controller's events.
  /// It gets a reference to the remove() callback, to delete the
  /// setState "link" from the Controller.
  /// 将controller绑定事件监听，传参会更新方法，根据是否传入id来判断是不是需要单独更新id对应的组件
  /// 同时返回_remove引用，用于在dispose()中解绑controller的监听链接
  /// 监听事件方法传参的getUpdate是GetStateUpdaterMixin的方法，最后实际还是调用了setState进行刷新
  void _subscribeToController() {
    _remove?.call();
    _remove = (widget.id == null)
        ? controller?.addListener(
            _filter != null ? _filterUpdate : getUpdate,
          )
        : controller?.addListenerId(
            widget.id,
            _filter != null ? _filterUpdate : getUpdate,
          );
  }

  void _filterUpdate() {
    var newFilter = widget.filter!(controller!);
    if (newFilter != _filter) {
      _filter = newFilter;
      getUpdate();
    }
  }

  @override
  void dispose() {
    super.dispose();
    widget.dispose?.call(this);
    if (_isCreator! || widget.assignId) {
      if (widget.autoRemove && GetInstance().isRegistered<T>(tag: widget.tag)) {
        GetInstance().delete<T>(tag: widget.tag);
      }
    }

    _remove?.call();
    ...
  }

  @override
  void didUpdateWidget(GetBuilder oldWidget) {
    super.didUpdateWidget(oldWidget as GetBuilder<T>);
    // to avoid conflicts when modifying a "grouped" id list.
    if (oldWidget.id != widget.id) {
      _subscribeToController();
    }
    widget.didUpdateWidget?.call(oldWidget, this);
  }

  @override
  Widget build(BuildContext context) {
    return widget.builder(controller!);
  }
}

mixin GetStateUpdaterMixin<T extends StatefulWidget> on State<T> {
  void getUpdate() {
    if (mounted) setState(() {});
  }
}
```

再看一下GetXController的源码

```dart
abstract class GetxController extends DisposableInterface
    with ListenableMixin, ListNotifierMixin {
  /// update方法接受ids参数，可以过滤需要刷新的GetBuilder id
  void update([List<Object>? ids, bool condition = true]) {
    if (!condition) {
      return;
    }
    if (ids == null) {
      ///调用ListNotifierMixin的方法
      refresh();
    } else {
      for (final id in ids) {
        ///调用ListNotifierMixin的方法
        refreshGroup(id);
      }
    }
  }
}
  @protected
  void refresh() {
    _notifyUpdate();
  }

  @override
  Disposer addListener(GetStateUpdate listener) {
    assert(_debugAssertNotDisposed());
    _updaters!.add(listener);
    return () => _updaters!.remove(listener);
  }

  void _notifyUpdate() {
    /// _updaters是GetBuilder注册监听时间的集合，里面存储了getUpdate和filterUpdate方法
    /// 遍历_updaters进行刷新
    for (var element in _updaters!) {
      element!();
    }
  }

  @protected
  void refreshGroup(Object id) {
    _notifyIdUpdate(id);
  }
```

GetBuilder的原理非常简单，最后我们总结一下：

    1、构造GetBuilder方法将View和Controller关联起来

    2、在GetBuilder方法中初始化Controller或者找到已经注册过的Controller实例

    3、对Controller添加刷新事件监听，将刷新发放添加到_updaters队列中

    4、需要刷新UI的时候调用Controller的update()方法，方法遍历_updaters队列，依次调用getUpdate或filterUpdate

    5、getUpdate或filterUpdate最后调用了setState方法刷新

### 3.3 响应式状态管理器（GetX）
响应式状态管理器顾名思义，只要响应式的变量发生变化时，界面和数据绑定的组件就会自动刷新，开发者无需再手动调用update()方法。

先看一下用法：
```dart
/// 在变量后面加上.obs,声明一个响应式变量
final name = ''.obs;
final isLogged = false.obs;
final count = 0.obs;
final balance = 0.0.obs;
final number = 0.obs;
final items = <String>[].obs;
final myMap = <String, int>{}.obs;

// 自定义类 - 可以是任何类
final user = User().obs;

Obx(()=> Text("Name ${name.value}: Age: ${count.value}"));
```

来看一下obs的解析
```dart
extension StringExtension on String {
  /// Returns a `RxString` with [this] `String` as initial value.
  RxString get obs => RxString(this);
}

extension IntExtension on int {
  /// Returns a `RxInt` with [this] `int` as initial value.
  RxInt get obs => RxInt(this);
}

extension DoubleExtension on double {
  /// Returns a `RxDouble` with [this] `double` as initial value.
  RxDouble get obs => RxDouble(this);
}

extension BoolExtension on bool {
  /// Returns a `RxBool` with [this] `bool` as initial value.
  RxBool get obs => RxBool(this);
}

extension RxT<T> on T {
  /// Returns a `Rx` instance with [this] `T` as initial value.
  Rx<T> get obs => Rx<T>(this);
}
```
.obs实际上是一个扩展，将变量用RxType的类包装起来，变成了响应式变量

再看一下Obx的类
```dart
/// Obx继承了ObxWidget类
class Obx extends ObxWidget {
  final WidgetCallback builder;

  const Obx(this.builder);

  @override
  Widget build() => builder();
}

abstract class ObxWidget extends StatefulWidget {
  const ObxWidget({Key? key}) : super(key: key);

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties..add(ObjectFlagProperty<Function>.has('builder', build));
  }

  @override
  _ObxState createState() => _ObxState();

  @protected
  Widget build();
}

class _ObxState extends State<ObxWidget> {
  final _observer = RxNotifier();
  late StreamSubscription subs;

  @override
  void initState() {
    super.initState();
    /// 绑定观察者订阅关系
    subs = _observer.listen(_updateTree, cancelOnError: false);
  }

  void _updateTree(_) {
    if (mounted) {
      setState(() {});
    }
  }

  @override
  void dispose() {
    subs.cancel();
    _observer.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) =>
      RxInterface.notifyChildren(_observer, widget.build);
}

static T notifyChildren<T>(RxNotifier observer, ValueGetter<T> builder) {
    final _observer = RxInterface.proxy;
    RxInterface.proxy = observer;
    final result = builder();
    if (!observer.canUpdate) {
      RxInterface.proxy = _observer;
      throw
      ...
    }
    RxInterface.proxy = _observer;
    return result;
  }
```
RxInterface.notifyChildren 把_observer作为RxInterface.proxy 的临时属性，在使用builder的后恢复原有的属性, 需要注意的是builder(controller)函数里一定要包含obs.value, 要不然会提示异常。

在从.value方法追进去看看
```dart
  T get value {
    RxInterface.proxy?.addListener(subject);
    return _value;
  }

  set value(T val) {
    if (subject.isClosed) return;
    sentToStream = false;
    if (_value == val && !firstRebuild) return;
    firstRebuild = false;
    _value = val;
    sentToStream = true;
    subject.add(_value);
  }

  void add(T event) {
    assert(!isClosed, 'You cannot add event to closed Stream');
    _value = event;
    _notifyData(event);
  }

  void _notifyData(T data) {
    _isBusy = true;
    for (final item in _onData!) {
      if (!item.isPaused) {
        item._data?.call(data);
      }
    }
    _isBusy = false;
  }
```
.value调用了getter方法，在RxInterface.proxy也就是_observer里添加了对该变量的订阅，当使用.value = xxx更新变量值的时候，就会调用setter方法，如果该被订阅对象已经失效或者被订阅对象的数值没有发生变化又或者已经被重新构建过，则跳过更新，否则将数值附上新的值，最终调用subject.add(_value),实质是调用了_notifyData方法，遍历onData的call去刷新UI。

_onData是通过上面_ObxState类中initState()方法里的listen调用添加的。

```dart

  StreamSubscription<T> listen(
    void Function(T) onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) =>
      subject.listen(
        onData,
        onError: onError,
        onDone: onDone,
        cancelOnError: cancelOnError ?? false,
      );

  LightSubscription<T> listen(void Function(T event) onData,
      {Function? onError, void Function()? onDone, bool? cancelOnError}) {
    final subs = LightSubscription<T>(
      removeSubscription,
      onPause: onPause,
      onResume: onResume,
      onCancel: onCancel,
    )
      ..onData(onData)
      ..onError(onError)
      ..onDone(onDone)
      ..cancelOnError = cancelOnError;
    addSubscription(subs);
    onListen?.call();
    return subs;
  }

  FutureOr<void> addSubscription(LightSubscription<T> subs) async {
    if (!_isBusy!) {
      return _onData!.add(subs);
    } else {
      await Future.delayed(Duration.zero);
      return _onData!.add(subs);
    }
  }

```


## 4 参考资料
* [flutter docs State management](https://docs.flutter.dev/development/data-and-backend/state-mgmt/intro)
* [GetX github 文档](https://github.com/jonataslaw/getx/blob/master/README.zh-cn.md)
* [Flutter GetX 使用 --- 简洁的魅力！](https://xie.infoq.cn/article/0ec29fcd60ac96f15aa0c3180)




