---
layout:     post                    # 使用的布局（不需要改）
title:      ConstraintLayout使用详解 上             # 标题 
subtitle:   ConstraintLayout使用详解  #副标题
date:       2018-11-22              # 时间
author:     2017-12-27                     # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签   
    - android

---
## 传统布局缺陷
![](https://img-blog.csdn.net/20171227223636859?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```html
<ScrollView>
    <LinearLayout>

		...
		...

        <LinearLayout>
        
            <LinearLayout>
                <LinearLayout/>
                <LinearLayout/>
                <LinearLayout/>
                <LinearLayout/>
            </LinearLayout>

            <LinearLayout>
                <LinearLayout/>
            </LinearLayout>

            <LinearLayout>
                <LinearLayout/>
                <LinearLayout/>
            </LinearLayout>
        </LinearLayout>
        
    </LinearLayout>
</ScrollView>
```
在这样场景下发现最多的时候用到四层线性布局,共嵌套了五层，即使使用RelativeLayout和LinearLayout结合使用的方式，最少也需要三层的布局，在面对复杂界面的时候，这样实现往往造成布局不够扁平，绘制性能也低下。
### 传统布局的缺点
* 复杂布局能力差，需要不同布局嵌套使用。
* 布局嵌套层级高。不同布局的嵌套使用，导致布局的嵌套层级偏高。
* 页面性能低。较高的嵌套层级，需要更多的计算布局时间，降低了页面性能。
* 按固定宽高比布局等更高阶的布局需求，原先的各类布局方式都不能很好的支持，可能需要通过Java代码，在运行中二次实现。

因此我们需要一个更加优雅的实现方式。

## ConstraintLayout的出现
### 在2016年Google I/O大会时提出，2017年2月发布正式版
A ConstraintLayout is a ViewGroup which allows you to position and size widgets in a flexible way.

Note: ConstraintLayout is available as a support library that you can use on Android systems starting with `API level 9` (Gingerbread). As such, we are planning on enriching its API and capabilities over time. This documentation will reflect those changes.

Android Studio虽然提供了可视化编写UI的功能，但是在ConstraintLayout出来之前，我认为它一直是鸡肋的存在，好无体验可言，开发过ios的都知道xCode 神器storyboard的便利之处，画UI几乎美工都可以完成，所见即所得，主要还是依赖了ios布局的大量的约束条件，ConstraintLayout的引入解决了Android界面编写的很多痛点，从名字就可以看出，约束布局通过丰富的约束条件可以达到降低页面布局层级、提升页面渲染性能的目的。精心使用完全可以替代其他布局。

## ConstraintLayout的使用
### 引入
* 在项目build.gradle文件中引入maven仓库

	```
repositories {
    	maven {
        	url 'https://maven.google.com'
    	}
}
	```

* 在APP build.gradle中依赖约束布局库

	`compile 'com.android.support.constraint:constraint-layout:1.0.2'`

* 在布局文件中ConstraintLayout要使用
 
	`xmlns:app="http://schemas.android.com/apk/res-auto"`

下面的介绍根据Android文档进行扩展
### Relative positioning 相对定位
Relative positioning is one of the basic building block of creating layouts in ConstraintLayout. Those constraints allow you to position a given widget relative to another one. You can constrain a widget on the horizontal and vertical axis:

* Horizontal Axis: left, right, start and end sides
* Vertical Axis: top, bottom sides and text baseline

属性名  	    				| 含义
------------------------- 	| -------------
layout_constraintLeft_toLeftOf 	| 左边缘和xx左边缘对齐
layout_constraintLeft_toRightOf	| 左边缘和xx右边缘对齐
layout_constraintRight_toLeftOf	| 右边缘和xx左边缘对齐
layout_constraintRight_toRightOf	| 右边缘和xx右边缘对齐
layout_constraintTop_toTopOf		| 上边缘和xx上边缘对齐
layout_constraintTop_toBottomOf	| 上边缘和xx下边缘对齐
layout_constraintBottom_toTopOf	| 下边缘和xx上边缘对齐
layout_constraintBottom_toBottomOf| 下边缘和xx下边缘对齐
layout_constraintBaseline_toBaselineOf| 基于baseline对齐
layout_constraintStart_toEndOf| 起始边缘和xx结束边缘对齐
layout_constraintStart_toStartOf| 起始边缘和xx起始边缘对齐
layout_constraintEnd_toStartOf| 结束边缘和xx起始边缘对齐
layout_constraintEnd_toEndOf| 结束边缘和xx结束边缘对其

嚯，一眼看上去ConstraintLayout的属性名也太长了吧，根本没法记啊，我一开始也是这么认为的，官方给出了解释：

**They all take a reference id to another widget, or the parent (which will reference the parent container, i.e. the ConstraintLayout)**

总结规律就是属性名里面**constraintxxx**就是我**当前控件的某条边缘**，**toxxxof**是我指定的**另一个布局或控件的某条边缘**，作用就是这两个边缘之间的相对关系。如果参照对象是parent布局，则可以完成和父布局边缘对齐的操作。例如layout_constraintLeft_toLeftOf = "parent"则是居左。

* 这里baseline做一个说明，baseline是文字的基线，可以参照某个控件(i.e. Button)中文字下边缘对齐。

	![这里写图片描述](http://img.blog.csdn.net/20171227223737376?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Margins 边距
属性和传统布局相同，这里就不多解释

属性名  | 
------------- | 
android:layout_marginStart|
android:layout_marginEnd |
android:layout_marginLeft|
android:layout_marginTop|
android:layout_marginRight|
android:layout_marginBottom|

### Margins when connected to a GONE widget 和 隐藏空间设置边距
When a position constraint target's visibility is View.GONE, you can also indicate a different margin value to be used using the following attributes:

可以和可见性已被设为View.GONE的控件设置边距，常用于在相对布局中保持各个控件的位置。因为我们在开发中常常遇到一个空间被设为GONE之后，和他有相对关联的控件的位置会发生改变，影响了原有的布局，导致UI混乱，用以下属性可以保持原有设计：

属性名  | 
------------- | 
layout_goneMarginStart|
layout_goneMarginEnd|
layout_goneMarginLeft|
layout_goneMarginTop|
layout_goneMarginRight|
layout_goneMarginBottom|

### Centering positioning and bias 居中定位和偏移

* center_vertical

```
 	app:layout_constraintTop_toTopOf="parent"
	app:layout_constraintBottom_toBottomOf="parent"
```

* center_horizontal

```
 	app:layout_constraintLeft_toLeftOf="parent"
	app:layout_constraintRight_toRightOf="parent"
```

* center

```
 	app:layout_constraintLeft_toLeftOf="parent"
	app:layout_constraintRight_toRightOf="parent"
	app:layout_constraintTop_toTopOf="parent"
	app:layout_constraintBottom_toBottomOf="parent"
```

* bias

	有人会说我线性布局一行代码实现居中，你这也太太太啰嗦了，naive，下面才是约束布局的新特性，一行代码实现固定比例偏移。比如下面这个场景，我要让A左右两边的空档宽度比例为3:7：
	
	![这里写图片描述](http://img.blog.csdn.net/20171227223948493?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

	在没了解ConstraintLayout之前，你一定会想到用LinearLayout配合layout_weight来完成：
	
	```
	    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="100dp"
        android:orientation="horizontal">

        <View
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="3"
            android:background="@color/colorAccent" />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="按钮" />

        <View
            android:layout_width="0dp"
            android:layout_height="match_parent"
            android:layout_weight="7"
            android:background="@color/colorAccent" />
    </LinearLayout>
    ```
    
    **一顿操作猛如虎，一看代码却很俗**
    
	怎么更加优雅的实现这样的场景呢，ConstraintLayout给出了答案：
    
	属性名  | 
------------- | 
layout_constraintHorizontal_bias|水平便宜比例
layout_constraintVertical_bias|竖直便宜比例
一行代码便可实现 666：

	```
	<android.support.constraint.ConstraintLayout ...>
	             <Button android:id="@+id/button" ...
	                 app:layout_constraintHorizontal_bias="0.3"
	                 app:layout_constraintLeft_toLeftOf="parent"
	                 app:layout_constraintRight_toRightOf="parent/>
	</>
	```
牛X的还不仅于此，收起下巴，下面的更强大

### Circular positioning (Added in 1.1) 1.1版本特性 圆弧定位(自己翻译）
You can constrain a widget center relative to another widget center, at an angle and a distance. This allows you to position a widget on a circle.

可以根据两个空间的中心位置形成约束关系，以当前组件中心为圆心设置半径和角度来设置约束条件

![这里写图片描述](http://img.blog.csdn.net/20171227224131215?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20171227224229929?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

属性名  	    				| 含义
------------------------- 	| -------------
layout_constraintCircle 	| 引用另一个控件id
layout_constraintCircleRadius	| 到xx中心位置的距离，即圆周半径
layout_constraintCircleAngle	| xx所在圆周的角度（0-360）

```
<Button android:id="@+id/buttonA" ... />
  <Button android:id="@+id/buttonB" ...
      app:layout_constraintCircle="@+id/buttonA"
      app:layout_constraintCircleRadius="100dp"
      app:layout_constraintCircleAngle="45" />
```
### Dimensions constraints 维度约束

* ConstraintLayout自身约束

 属性名  	    				| 含义
------------------------- 	| -------------
android:minWidth| set the minimum width for the layout
android:minHeight| set the minimum height for the layout
android:maxWidth |set the maximum width for the layout
android:maxHeight |set the maximum height for the layout

**注意上述属性设置ConstraintLayout本身最大最小尺寸只适用于自身为WRAP_CONTENT下**

### Percent dimension 百分比维度
可以设置控件相对于父布局尺寸的百分比，0-1之间。需要注意：

* 控件的width和height要被设置为0dp
* 默认宽高属性要被设置为percent(1.1版本之后将无需设置，自动识别）

 ```
app:layout_constraintWidth_default="percent"
app:layout_constraintHeight_default="percent" 
 ```
```
 <Button
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintWidth_percent="0.3"
        app:layout_constraintHeight_percent="0.2"
        />
```
设置按钮的宽度为父布局宽度的30%，高度为父布局高度的20%。


### Ratio 比例
可以定义一个控件宽高尺寸比例，这样可以固定控件的尺寸关系，形成一种约束，减少出现UI混乱被挤压的情况，可以形成自适应。但是要注意的事，控件的两个维度height和width中至少要有一个设置为0或者MATCH_CONSTRAINT，这个很好理解，不解释。

```
<Button android:layout_width="wrap_content"
        android:layout_height="0dp"
        app:layout_constraintDimensionRatio="1:1" />
```
这样就可以不用指定一个正方形的宽高，设计合理的话可以根据屏幕自适应。

同样，也可以只限制其中一个维度（height或width）随另一个维度的变化的比例。
如果一个维度固定，只需要在比例前面加上**“H"**或者**”W"**表示想要对哪个维度进行限制。

```
 <Button android:layout_width="0dp"
         android:layout_height="0dp"
         app:layout_constraintDimensionRatio="H,16:9"
         app:layout_constraintBottom_toBottomOf="parent"
         app:layout_constraintTop_toTopOf="parent"/>
```
上面代码中按钮的width将会充满父布局，而高度则会以16:9的比例进行测量绘制。

### Chains 链
chain是ConstraintLayout的新概念，可以让多个组件在一个方向上（水平或竖直）组合成一个集体，官方称 **provide group-like behavior**。
我们知道，以往的控件之间的相对联系是一个控件A基于另一个控件B的位置关联，控件A随着B的变动变动，反之则不行，但是Chain的存在，可以说是一个组合，让多个控件组成了一个集体，一直保持固定的相对关系，这样做的好处是可以减少布局兼容时造成的混乱，方便整体复用。

* 创建一个链约束

  链约束的关键在于需要组合在一起的布局之间在**同一个方向**上是**双向连接**，也就是相互进行相对约束。

  ![这里写图片描述](http://img.blog.csdn.net/20171227224343331?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  对于 A、B两个button，形成链约束代码如下：

  ```xml
  <Button
          android:id="@+id/btn1"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:text="A"
          app:layout_constraintLeft_toLeftOf="parent"
          app:layout_constraintRight_toLeftOf="@id/btn2" />

  <Button
          android:id="@+id/btn2"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:text="B"
          app:layout_constraintLeft_toRightOf="@id/btn1"
          app:layout_constraintRight_toRightOf="parent" />
  ```

  ![这里写图片描述](http://img.blog.csdn.net/20171227224532318?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* chain heads 链头

  链是由链上第一个元素（链头）上的属性集控制，链头是最左边或最上边的组件。

  ![这里写图片描述](http://img.blog.csdn.net/20171227224602589?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



* chain style 链内布局方式

  **layout_constraintHorizontal_chainStyle** 水平链内布局方式

  **layout_constraintVertical_chainStyle** 竖直链内布局方式

  只要在**链头**组件设置chain style，便可以控制整条链的内部组件排列布局方式。

  **默认style为CHAIN_SPREAD**，即组件间隔均分可使用空间。

  ![这里写图片描述](http://img.blog.csdn.net/20171227224629498?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvS2luc29teQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  ​

## 性能比较

由于ConstraintLayout天生扁平的优点，在性能上肯定要优于传统布局不少，关于性能对比的详细情况，可以查看这篇文章[了解使用 ConstraintLayout 的性能优势](http://developers.googleblog.cn/2017/09/constraintlayout.html)


## 布局编辑器（待更新）
## 代码设置布局（待更新）
## 源码解析（待更新）


## 参考文献

1、 [官方文档 ](https://developer.android.com/reference/android/support/constraint/ConstraintLayout.html "Title")

2、[Android新特性介绍，ConstraintLayout完全解析](http://blog.csdn.net/guolin_blog/article/details/53122387 "Title")  郭霖

3、[ConstraintLayout入门指南](https://mp.weixin.qq.com/s/X01KpEbegR47Qnl9TmUd5w) QQ音乐技术团队

4、[ConstraintLayout在项目中实践与总结](https://juejin.im/post/5a1d9ba66fb9a044fb07819e#heading-26) 

5、[了解使用 ConstraintLayout 的性能优势](http://developers.googleblog.cn/2017/09/constraintlayout.html)
