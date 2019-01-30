---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(二)View的测量、布局、绘制流程          # 标题 
subtitle:   Android advance #副标题
date:       2018-07-06            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---


## 1 引言
在上一篇文章[Android进阶(一)View体系](https://kinsomyjs.github.io/2019/01/29/Android%E8%BF%9B%E9%98%B6(%E4%B8%80)View%E4%BD%93%E7%B3%BB/)中，分析了Android源码关于activity启动创建view的过程，在`WindowManagerGlobal`的addView方法里面调用了`ViewRootImpl`构造方法，构造root，同时在ViewRootImpl里面会调用一个`performTraversals()`方法，看一下源码：

```java
private void performTraversals() {
    ....
    if (!mStopped || mReportNextDraw) {
        boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                (relayoutResulWindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
        if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                || mHeight != host.getMeasuredHeight() |contentInsetsChanged ||
                updatedConfiguration) {
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeightlp.height)
            if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!mWidth="
                    + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                    + " mHeight=" + mHeight
                    + " measuredHeight=" + host.getMeasuredHeight()
                    + " coveredInsetsChanged=" + contentInsetsChanged)
             // Ask host how big it wants to be 注释①
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); 
            ...
            }
    }  
    ...
    if (didLayout) {
        //注释②
        performLayout(lp, mWidth, mHeight);
        ...
    }
    ...
    if (!cancelDraw && !newSurface) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }
            //注释③
            performDraw();
        }
}
```
可以看到在注释①②③分别会执行`performMeasure`,`performLayout`以及`performDraw`方法,它们会分别调用mView的measure，layout和draw方法，这里的mView就是从`WindowManagerGlobal`的setView方法里传递进来的DecorView，所以也就是在这里会对DecorView进行`测量、布局和绘制`,这三个方法会有三个熟悉的回调方法，就是`onMeasure`、`onLayout`和`onDraw`,可供用户自定义，完成之后，用户就能在activity内看到界面了。

## 2 Measure
我们先来看看Measure流程，看performMeasure方法源码：
```java
    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
mView的measure方法会有两个参数：childWidthMeasureSpec，childHeightMeasureSpec，看名字能看出来和view的宽高有关系，追踪到该方法使用的地方，可以看到这里的两个参数则是由如下代码传递过来：
```java
if (baseSize != 0 && desiredWindowWidth > baseSize) {
    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
    childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}

/**
 * Figures out the measure spec for the root view in a window based on it's
 * layout params.
 *
 * @param windowSize
 *            The available width or height of the window
 *
 * @param rootDimension
 *            The layout params for one dimension (width or height) of the
 *            window.
 *
 * @return The measure spec to use to measure the root view.
 */
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize,MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize,MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension,MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```
所以childWidthMeasureSpec和childHeightMeasureSpec都是通过`getRootMeasureSpec(int windowSize, int rootDimension)`获得的，这个方法注释说明它是用来为rootview根据window和layout params的值来计算得出measure spec，我们都知道，一个view的尺寸是由自身尺寸和父布局尺寸共同决定的，因为这里的view是根布局DecorView，它的父布局就是整个window，所以它的尺寸是由自身和window尺寸决定。那这里的`MeasureSpec`类又是什么呢？看一下类的注释：
```java
    /**
     * A MeasureSpec encapsulates the layout requirements passed from parent to child. Each MeasureSpec represents a requirement for either the width or the height. A MeasureSpec is comprised of a size and a mode. There are three possible modes:
     * <dt>UNSPECIFIED</dt>
     * The parent has not imposed any constraint on the child. It can be whatever size it wants.
     * <dt>EXACTLY</dt>
     * The parent has determined an exact size for the child. The child is going to be given those bounds regardless of how big it wants to be.
     * <dt>AT_MOST</dt>
     * The child can be as large as it wants up to the specified size.
     * MeasureSpecs are implemented as ints to reduce object allocation. This class is provided to pack and unpack the &lt;size, mode&gt; tuple into the int.
     */
```
MeasureSpec就是组成了一个包括size和SpecMode的类，size就是指的宽高尺寸，mdoe分为三种
* UNSPECIFIED 父布局对子布局没有任何限制，子布局可以任意尺寸
* EXACTLY 父布局给了子布局精确的尺寸限制，子布局需要在边界内 对应于match_parent
* AT_MOST 子布局可以最大化到父布局指定的尺寸 对应于wrap_content。

view的measure分为View和ViewGroup两种，View只需要根据measure给定的specsize即可，而ViewGroup除了要测量自身，还需要对子布局调用measure方法，指定他们的size。

### 2.1 view的measure流程
看一下view的onMeasure方法：
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(),widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

protected final void setMeasuredDimension(int measuredWidth, intmeasuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;
        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}

protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
}
```
`onMeasure`首先会调用`setMeasuredDimension`,传入`getDefaultSize`返回的size，`getDefaultSize`是根据不同的specmode返回不同的view尺寸，size是view指定的尺寸，specSize则是根据specmode返回的size，AT_MOST和EXACTLY返回的result是相同的。

再来看`getSuggestedMinimumWidth`,如果view没有背景就会返回view的minWidth，如果有背景，就会返回view和背景的最小宽度中的较大的那个。

### 2.2 ViewGroup的measure流程
上面说到了ViewGroup不仅要测量自身尺寸，还要用measure方法给每个子布局指定尺寸。然而ViewGroup和单个View不同，他的布局相对复杂多变，ViewGroup并不知道它的子类会怎么样进行布局，因此也就不能指定onMeasure方法，而是让它的子类重写自行测量，但是实现了一个`measureChildren`方法,依次遍历每个子View，给他们单独设置尺寸。
```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
## 3 Layout
和Measure类似，layout在View和ViewGroup中也不相同，View中的layout方法用来确定自身的位置,ViewGroup则是来指定子布局的位置。

看一下View的layout方法：
```java
public void layout(int l, int t, int r, int b) {
    ...
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = isLayoutModeOptical(mParent) ?
        setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
            ...
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }   
    }
}
```
layout的四个参数是当前view距离父布局的四边缘的相对距离，old的四个变量是如果布局位置改变保存的上一个位置的距离父布局四边缘的相对距离，当调用layout方法设置布局位置的时候，会回调onLayout()方法。onLayout方法的第一个参数是通过setFrame方法和上一次记录的位置比较是否有变化返回的。

## 4 Draw
Draw就是进行view的绘制工作，看一下view的draw方法：
```java
/**
 * Manually render this view (and all of its children) to the given Canvas.
 * The view must have already done a full layout before this function is
 * called.  When implementing a view, implement
 * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
 * If you do need to override this method, call the superclass version.
 *
 * @param canvas The Canvas to which the View is rendered.
 */
@CallSuper
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */
```
从注释上就可以知道首先draw方法被调用之前layout必须是被完整执行了，当需要绘制view的时候，是重写`onDraw`方法，而不需要重写draw方法。绘制需要按照以下顺序执行：
* 1、绘制背景
* 2、如果需要渐变，则保存canvas层
* 3、绘制view的内容
* 4、绘制view的子布局
* 5、如果需要，绘制view的渐变边缘
* 6、绘制例如滚动条的装饰器

每一步的具体操作在方法里都有注释，这里就不一一讲述了，记住需要绘制view的话需要重写onDraw。

## 参考资料
* 《Android进阶之光》
* [View 体系详解：View 的工作流程](https://shouheng88.github.io/2018/10/14/View%20%E4%BD%93%E7%B3%BB%E8%AF%A6%E8%A7%A3%EF%BC%9AView%20%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B/)

