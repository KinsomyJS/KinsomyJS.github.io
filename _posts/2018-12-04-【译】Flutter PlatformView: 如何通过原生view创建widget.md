---
layout:     post                    # 使用的布局（不需要改）
title:   Flutter Platform:如何通过原生view创建widget                # 标题 
subtitle:   Flutter PlatformView  #副标题
date:       2018-12-04              # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - flutter
    - dart
    - android
    
---
**原文来自 flutter community**

https://medium.com/flutter-community/flutter-platformview-how-to-create-flutter-widgets-from-native-views-366e378115b6)

### 原由
最近想要实现一个flutter地图插件，但是flutter只提供了基于google map的官方插件，所以想要使用高德地图和百度地图自己做一个，就看到了这篇文章，顺便翻译一下分享给大家。

这里先贴出google map插件文章以便有梯子的同学可以直接使用
[Exploring Google Maps in Flutter](https://medium.com/flutter-community/exploring-google-maps-in-flutter-8a86d3783d24)

### 译文
![](https://user-gold-cdn.xitu.io/2018/12/4/167785ab29236f51?w=800&h=480&f=png&s=44430)

Flutter最近刚刚有了一个新的组件叫做`PlatformView`，它允许开发者在flutter里面嵌入Android原生的view。这一开放举措为实现注入地图和webview提供了许多新的可能。(https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/widgets/platform_view.dart).

在这篇教程里，我们将探索如何创建一个`TextViewPlugin`，在plugin里我们会暴露一个android原生TextView作为flutter组件。

在进入代码实现之前需要注意以下几点：
* 目前只支持Android（作者文章发布于2018.9.7，**目前已经支持ios**）。
* 需要android api版本在20及以上。
* 嵌入Android views是一个昂贵的操作，所以应当避免在flutter能够实现的情况下去使用它。
* 嵌入Android view的绘制和其他任何flutter widget一样，view的转换也同样使用。
* 组件会撑满所有可获得控件，因此它的父组件需要提供一个布局边界。
* 需要切换到flutter的master分支上使用（目前已经不需要）

下面开始实现部分

第一步就是创建一个flutter plugin项目。

![](https://user-gold-cdn.xitu.io/2018/12/4/167786e430da20f8?w=800&h=525&f=png&s=85924)

![](https://user-gold-cdn.xitu.io/2018/12/4/167786ed1b5c1316?w=800&h=669&f=png&s=62730)

在flutter plugin项目创建完成之后在`./lib/text_view.dart`创建TextView类。

```dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

typedef void TextViewCreatedCallback(TextViewController controller);

class TextView extends StatefulWidget {
  const TextView({
    Key key,
    this.onTextViewCreated,
  }) : super(key: key);

  final TextViewCreatedCallback onTextViewCreated;

  @override
  State<StatefulWidget> createState() => _TextViewState();
}

class _TextViewState extends State<TextView> {
  @override
  Widget build(BuildContext context) {
    if (defaultTargetPlatform == TargetPlatform.android) {
      return AndroidView(
        viewType: 'plugins.felix.angelov/textview',
        onPlatformViewCreated: _onPlatformViewCreated,
      );
    }
    return Text(
        '$defaultTargetPlatform is not yet supported by the text_view plugin');
  }

  void _onPlatformViewCreated(int id) {
    if (widget.onTextViewCreated == null) {
      return;
    }
    widget.onTextViewCreated(new TextViewController._(id));
  }
}

class TextViewController {
  TextViewController._(int id)
      : _channel = new MethodChannel('plugins.felix.angelov/textview_$id');

  final MethodChannel _channel;

  Future<void> setText(String text) async {
    assert(text != null);
    return _channel.invokeMethod('setText', text);
  }
}

```
一个需要重点注意的是当我们在上面代码第24行创建了`AndroidView`，我们需要提供一个`viewType`,会在稍后介绍。

我们还提供了一个`onPlatformCompleted`实现,以便我们可以为`TextView`小部件提供一个`TextViewController`，然后可以使用它来使用`setText`方法更新它的文本。

完整的`AndroidView`文档请见 https://docs.flutter.io/flutter/widgets/AndroidView-class.html

接下来我们需要实现Android部分的`TextViewPlugin`。

我们打开另一个生成文件`./android/src/main/java/{organization_name}/TextViewPlugin.java`,用一下内容替换文件内的内容。

```java
package angelov.felix.textview;

import io.flutter.plugin.common.PluginRegistry.Registrar;

public class TextViewPlugin {
  public static void registerWith(Registrar registrar) {
    registrar
            .platformViewRegistry()
            .registerViewFactory(
                    "plugins.felix.angelov/textview", new TextViewFactory(registrar.messenger()));
  }
}
```

我们需要做的就是实现`registerWith`方法，传入`text_view.dart`中定义的`viewType`，同时提供一个`TextViewFactory`，它将会创建原生`TextView`作为`PlatformView`。

接着我们需要创建`./android/src/main/java/{organization_name}/TextViewFactory.java`，并继承`PlatformViewFactory`

```java
package angelov.felix.textview;

import android.content.Context;
import io.flutter.plugin.common.BinaryMessenger;
import io.flutter.plugin.common.StandardMessageCodec;
import io.flutter.plugin.platform.PlatformView;
import io.flutter.plugin.platform.PlatformViewFactory;

public class TextViewFactory extends PlatformViewFactory {
    private final BinaryMessenger messenger;

    public TextViewFactory(BinaryMessenger messenger) {
        super(StandardMessageCodec.INSTANCE);
        this.messenger = messenger;
    }

    @Override
    public PlatformView create(Context context, int id, Object o) {
        return new FlutterTextView(context, messenger, id);
    }
}
```
`TextViewFactory`实现了create方法，它会返回`PlatformView`，在我们的例子里叫做`FlutterTextView`。

然后，创建`./android/src/main/java/{organization_name}/FlutterTextView.java`并且实现`PlatformView`和`MethodCallHandler`以至于我们可以将原生view绘制到flutter组件并且通过`MethodChannel`从dart接收数据。
```java
package angelov.felix.textview;

import android.content.Context;
import android.view.View;
import android.widget.TextView;
import io.flutter.plugin.common.MethodCall;
import io.flutter.plugin.common.MethodChannel;
import static io.flutter.plugin.common.MethodChannel.MethodCallHandler;
import static io.flutter.plugin.common.MethodChannel.Result;
import io.flutter.plugin.common.BinaryMessenger;
import io.flutter.plugin.platform.PlatformView;

public class FlutterTextView implements PlatformView, MethodCallHandler  {
    private final TextView textView;
    private final MethodChannel methodChannel;

    FlutterTextView(Context context, BinaryMessenger messenger, int id) {
        textView = new TextView(context);
        methodChannel = new MethodChannel(messenger, "plugins.felix.angelov/textview_" + id);
        methodChannel.setMethodCallHandler(this);
    }

    @Override
    public View getView() {
        return textView;
    }

    @Override
    public void onMethodCall(MethodCall methodCall, MethodChannel.Result result) {
        switch (methodCall.method) {
            case "setText":
                setText(methodCall, result);
                break;
            default:
                result.notImplemented();
        }

    }

    private void setText(MethodCall methodCall, Result result) {
        String text = (String) methodCall.arguments;
        textView.setText(text);
        result.success(null);
    }

    @Override
    public void dispose() {}
}

```

`FlutterTextView`创建了一个新的`TextView`并且设置了一个`MethodChannel`，因此TextView可以从dart代码接收数据进行视图更新。

为了实现`PlatformView`,我们需要重写`getView`方法返回textview对象，同时重写`dispose`方法。

为了实现`MethodCallHandler`，需要重写`onMethodCall`，在接收到**setText**调用指令时候更新TextView，或者在不支持的其他指令情况下返回`result.notImplemented`。

现在可以测试我们的新`TextView`组件。

打开`./example/lib/main.dart`用下面的代码替换。
```dart
import 'package:flutter/material.dart';
import 'package:text_view/text_view.dart';

void main() => runApp(MaterialApp(home: TextViewExample()));

class TextViewExample extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
        appBar: AppBar(title: const Text('Flutter TextView example')),
        body: Column(children: [
          Center(
              child: Container(
                  padding: EdgeInsets.symmetric(vertical: 30.0),
                  width: 130.0,
                  height: 100.0,
                  child: TextView(
                    onTextViewCreated: _onTextViewCreated,
                  ))),
          Expanded(
              flex: 3,
              child: Container(
                  color: Colors.blue[100],
                  child: Center(child: Text("Hello from Flutter!"))))
        ]));
  }

  void _onTextViewCreated(TextViewController controller) {
    controller.setText('Hello from Android!');
  }
}

```
代码看着很熟悉，我们运行一个`MaterialApp`,最外层用`Scaffold`。有一个`TextView`用`Container`包裹，这个container组件作为一个垂直排列组件的第一个子组件，第二个子组件是一个flutter `Text`。

我们同样实现了方法`onTextViewCreated`，在这个方法里面去调用`setText`。


![](https://user-gold-cdn.xitu.io/2018/12/4/167788dd68cfa0cc?w=676&h=1408&f=png&s=141252)

这个例子证明了可以对任何原生Android view转换成flutterwidget，并且像其他flutter组件一样绘制和转换。