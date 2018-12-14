---
layout:     post                    # 使用的布局（不需要改）
title:   深入Flutter TextField                # 标题 
subtitle:   Flutter TextField  #副标题
date:       2018-12-14              # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - flutter
---

翻译原文来自 https://medium.com/flutter-community/a-deep-dive-into-flutter-textfields-f0e676aaab7a

## TextField介绍
TextField允许从用户处收集信息，声明一个基础的TextField代码十分简单：
```dart
TextField()
```
这会创建一个基础TextField组件：

![](https://user-gold-cdn.xitu.io/2018/12/13/167a6e566303a4cb?w=760&h=110&f=png&s=676)

## 从TextField收集信息
由于TextFields没有像Android中那样有一个ID，因此无法根据需要检索文本，而必须在更改时将其存储在变量中或使用控制器。

1.最简单的方式就是使用`onChanged`回调方法，将TextField当前的值保存在一个变量内，下面是示例代码。
```dart
String value = "";
TextField(
  onChanged: (text) {
    value = text;
  },
)
```

2.第二种方式就是使用`TextEditingController`。控制器依附在TextField组件上，并且我们可以监听和控制TextField的文本。
```dart
TextEditingController controller = TextEditingController();
TextField(
  controller: controller,
)
```

接着就可以使用下面代码监听变化：
```dart
controller.addListener(() {
  // Do something here
});
```

用控制器获取和设置文本：
```dart
print(controller.text); // Print current value
controller.text = "Demo Text"; // Set new value
```

## TextField的其他回调
```
//键盘上按了done
onEditingComplete: () {},
onSubmitted: (value) {},
```

## TextField获得焦点
TextField获得焦点意味着有一个TextField被激活并且任何来自键盘的输入都会被键入到获得焦点的TextField内。

### 1.自动获得焦点
为了让TextField在widget创建的时候自动获取焦点，设置autofocus属性为true。
```dart
TextField(
  autofocus: true,
),
```

![](https://user-gold-cdn.xitu.io/2018/12/13/167a6f2cbdcd3646?w=1080&h=1920&f=gif&s=799245)

### 2.自定义焦点切换
如果我们想代码变化焦点而不是仅仅自动获得焦点需要怎么做呢？由于我们需要某种方式来引用我们想要关注的TextField，我们将FocusNode附加到TextField并使用它来切换焦点。

```dart
// Initialise outside the build method
FocusNode nodeOne = FocusNode();
FocusNode nodeTwo = FocusNode();
// Do this inside the build method
TextField(
  focusNode: nodeOne,
),
TextField(
  focusNode: nodeTwo,
),
RaisedButton(
  onPressed: () {
    FocusScope.of(context).requestFocus(nodeTwo);
  },
  child: Text("Next Field"),
),
```
这里我们创建了两个focus node并且将他们依附到TextField上，当点击NextField按钮时，使用`FocusScope`去为下一个TextField申请获取焦点。

![](https://user-gold-cdn.xitu.io/2018/12/13/167a6f74ab0f78de?w=1080&h=1920&f=gif&s=392827)

## TextField更换键盘属性
在Flutter中，TextField允许你定制和键盘相关的属性。
### 1.键盘类型
TextField可以在弹出键盘的时候修改键盘类型。使用以下代码：
```dart 
TextField(
  keyboardType: TextInputType.number,
),
```

类型有如下几种：
* TextInputType.text (Normal complete keyboard)
* TextInputType.number (A numerical keyboard)
* TextInputType.emailAddress (Normal keyboard with an “@”)
* TextInputType.datetime (Numerical keyboard with a “/” and “:”)
* TextInputType.multiline (Numerical keyboard with options to enabled signed and decimal mode)

### 2.键盘操作按钮行为
更改TextField的textInputAction可以更改键盘本身的操作按钮。

```dart
//发送操作
TextField(
  textInputAction: TextInputAction.send,
),
```

![](https://user-gold-cdn.xitu.io/2018/12/13/167a725651b92c7d?w=762&h=434&f=png&s=62802)

完整的行为列表太长，这里不做展示了，需要的可以进入TextInputAction类查看。

启用或禁用特定TextField的自动更正。使用自动更正字段进行设置。这同时也会禁用输入建议。
```dart
TextField(
  autocorrect: false,
),
```

### 3.文本大写
TextField提供了一些选项，用来对用户输入的文本大写化。
```dart
TextField(
  textCapitalization: TextCapitalization.sentences,
),
```
#### 1.TextCapitalization.sentences
这是最常见的大写化类型，每个句子的第一个字母被转换成大写。

![](https://user-gold-cdn.xitu.io/2018/12/14/167aa6e5c845bfd2?w=748&h=152&f=png&s=9879)

#### 2.TextCapitalization.characters
大写句子中的所有字符。

![](https://user-gold-cdn.xitu.io/2018/12/14/167aa6fcde9f9335?w=756&h=136&f=png&s=7662)

#### 3. TextCapitalization.words
对每个单词首字母大写。

![](https://user-gold-cdn.xitu.io/2018/12/14/167aaaeddea9a436?w=748&h=134&f=png&s=5448)

### 文本对齐
使用textAlign属性设置TextField内的光标对齐方式。
```dart
TextField(
  textAlign: TextAlign.center,
),
```
光标和文字会从TextField组件中间开始。

![](https://user-gold-cdn.xitu.io/2018/12/14/167aab23dd8c4203?w=758&h=138&f=png&s=2858)
还有其他属性包括，**start, end, left, right, center, justify.**

### 文本样式
我们使用style属性来更改TextField内部文本的外观。
使用它来更改颜色，字体大小等。这类似于文本小部件中的样式属性，因此我们不会花太多时间来探索它。

```dart
TextField(
  style: TextStyle(color: Colors.red, fontWeight: FontWeight.w300),
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aabc797ec2d3d?w=756&h=164&f=png&s=4560)

### 更换光标样式
可以直接从TextField小部件自定义光标。
您可以更改角落的光标颜色，宽度和半径。
例如，这里我制作一个圆形的红色光标。

```dart
TextField(
  cursorColor: Colors.red,
  cursorRadius: Radius.circular(16.0),
  cursorWidth: 16.0,
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aabdf059ce83e?w=758&h=150&f=png&s=1679)

### 控制最大字符数
```dart
TextField(
  maxLength: 4,
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aabf98508bf56?w=754&h=158&f=png&s=4444)
设置maxLength属性后，TextField会默认添加一个长度计数器。

### 可扩展TextField
有时，我们需要一个TextField，当一行完成时会扩展。
在Flutter中，它有点奇怪（但很容易）。
为此，我们将maxLines设置为null，默认为1。
设置为null不是我们习以为常的东西，但是它很容易做到。

**注意：将maxLines属性设置为一个确定的数值，会将TextField直接扩大到对应的最大行**
```dart
TextField(
  maxLines: 3,
)
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aac49b8fe3d26?w=756&h=200&f=png&s=982)

### 隐藏文本内容
```dart
TextField(
  obscureText: true,
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aac64ec301126?w=744&h=142&f=png&s=6183)

## 装饰TextField
到目前为止，我们专注于Flutter提供的输入功能。
现在我们将实际设计一个花哨的TextField。
为了装饰TextField，我们使用带有InputDecoration的decoration属性。
由于InputDecoration类非常庞大，我们将尝试快速查看大多数重要属性。

### hint和label

hint:
![](https://user-gold-cdn.xitu.io/2018/12/14/167aaca7902871f9?w=748&h=124&f=png&s=4649)

label:
![](https://user-gold-cdn.xitu.io/2018/12/14/167aaca93e13300b?w=742&h=152&f=png&s=3888)

### 图标
```dart
//输入框外图标
TextField(
  decoration: InputDecoration(
    icon: Icon(Icons.print)
  ),
),
//前缀图标
TextField(
  decoration: InputDecoration(
    prefixIcon: Icon(Icons.print)
  ),
),

//输入框前缀组件
TextField(
  decoration: InputDecoration(
    prefix: CircularProgressIndicator(),
  ),
),
```


![](https://user-gold-cdn.xitu.io/2018/12/14/167aacd8d2ffefb6?w=756&h=148&f=png&s=1385)

![](https://user-gold-cdn.xitu.io/2018/12/14/167aacda35ffa409?w=748&h=180&f=png&s=1472)

![](https://user-gold-cdn.xitu.io/2018/12/14/167aacdbca108370?w=756&h=194&f=png&s=7000)

### 每个属性像hint，label都有各自的style设置
#### 1.用hintstyle修改hint样式
```dart
TextField(
  decoration: InputDecoration(
    hintText: "Demo Text",
    hintStyle: TextStyle(fontWeight: FontWeight.w300, color: Colors.red)
  ),
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aad24e328dc6c?w=748&h=142&f=png&s=4835)

#### 2.帮助性提示
和label不一样，它会一直显示在输入框下部
```dart
TextField(
  decoration: InputDecoration(
    helperText: "Hello"
  ),
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167aad3cffc7a844?w=748&h=166&f=png&s=2038)

#### 3.使用decoration: null或者InputDecoration.collapsed可以去除默认的下划线。
```dart
TextField(
  decoration: InputDecoration.collapsed(hintText: "")
),
```

#### 4.使用border给TextField设置边框
```dart
TextField(
  decoration: InputDecoration(
    border: OutlineInputBorder()
  )
),
```

![](https://user-gold-cdn.xitu.io/2018/12/14/167ab047cf350e9f?w=754&h=174&f=png&s=1543)