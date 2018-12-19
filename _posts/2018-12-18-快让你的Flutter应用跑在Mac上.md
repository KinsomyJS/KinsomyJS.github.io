---
layout:     post                    # 使用的布局（不需要改）
title:   快让你的Flutter应用跑在Mac上            # 标题 
subtitle:   Feather platform    #副标题
date:       2018-12-18              # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - flutter
---

## 介绍
![](https://user-gold-cdn.xitu.io/2018/12/18/167c03bf52b098a3?w=867&h=248&f=png&s=30106)

今天介绍一个能让Flutter应用运行在Mac OS和Windows上的平台：**Feather Platform**

官网地址：https://feather-apps.com/

官网的介绍如下：

The Feather platform will run Flutter apps on MacOS and Windows. So you can write a single app that runs on all major desktop and mobile devices.

基本可以说写一次flutter app，可以在全平台运行了。

那具体要怎么操作呢？

## 实践

#### 1.在官网首页点击按钮 **Build an App Now**,会下载程序安装包。

![](https://user-gold-cdn.xitu.io/2018/12/18/167c040993d12592?w=1037&h=542&f=png&s=711040)

#### 2.下载之后打开安装应用,就进入了如下的应用界面。

![](https://user-gold-cdn.xitu.io/2018/12/18/167c041cf41f842b?w=517&h=678&f=png&s=26886)

#### 3.用谷歌账号登录（需要科学上网）。
![](https://user-gold-cdn.xitu.io/2018/12/18/167c043168ecdc73?w=217&h=378&f=png&s=26115)

#### 4.点击右下角“添加”按钮。

![](https://user-gold-cdn.xitu.io/2018/12/18/167c04426116b3ff?w=517&h=678&f=png&s=35751)

#### 5.点击"BROWSE"选择一个已经开发完成的Flutter项目。
这里我们用MusesWeather项目做实验。

项目地址：https://github.com/KinsomyJS/muses_weather_flutter

文章地址：https://juejin.im/post/5bc43027f265da0ad221b5e5

#### 6.增加代码

![](https://user-gold-cdn.xitu.io/2018/12/18/167c0479f1a95ca6?w=436&h=443&f=png&s=37299)

按照提示需要增加两处代码，在项目的main.dart文件添加`import 'package:flutter/foundation.dart';`,
在main方法里添加`debugDefaultTargetPlatformOverride = TargetPlatform.iOS;`。

#### 7.添加App Name并点击Continue
![](https://user-gold-cdn.xitu.io/2018/12/18/167c04a51919a956?w=517&h=678&f=png&s=37726)

这样就得到了一个添加好了的App项目，点击进去会看到

![](https://user-gold-cdn.xitu.io/2018/12/18/167c04b47d55ed62?w=517&h=678&f=png&s=38647)

点击TEST就会提示你打开Xcode，然后在Xcode里面run 工程。

![](https://user-gold-cdn.xitu.io/2018/12/18/167c04ea12515694?w=517&h=678&f=png&s=65716)


### 产品
最后我们就成功将写好的Flutter 项目运行在了Mac OS上，感兴趣的同学可以立马尝试下。

项目地址：https://github.com/KinsomyJS/muses_weather_flutter


![](https://user-gold-cdn.xitu.io/2018/12/18/167c0591cc27d83f?w=600&h=375&f=gif&s=4584133)

### 解释
How is this different to the flutter-desktop-embedding project?

Feather is actually based on the flutter-desktop-embedding project. Currently for Mac it offers the same features plus:

(a) More functionality like copy and paste, mouse wheel and escape key

(b) More supported plugins like shared_preferences, url_launcher, google_sign_in 

(c) An easy way to publish your app and push updates to end users

其实Feather Platform就是在Google开源项目[flutter-desktop-embedding](https://github.com/google/flutter-desktop-embedding/)的基础上开发的，并提供了更多的特性：
* 键盘和鼠标等输入设备
* 支持更多插件如持久化，google登录等
* 可以发布app到Feather商店并且更新。