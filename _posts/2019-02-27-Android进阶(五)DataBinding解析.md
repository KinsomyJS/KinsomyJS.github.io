---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(五)DataBinding解析          # 标题 
subtitle:   Android advance #副标题
date:       2019-02-27            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---
## 1 概述
在上篇文章[Android进阶(四)LiveData解析](https://kinsomyjs.github.io/2019/02/25/Android%E8%BF%9B%E9%98%B6(%E5%9B%9B)LiveData%E8%A7%A3%E6%9E%90/)中讲到了关于JetPack框架的LiveData解析，这是一个基于ViewModel和观察者模式的实践。

这篇文章要讲的DataBinding同样可以认为是基于ViewModel的实践，同时做到了数据和UI的双向绑定。DataBinding允许你使用声明式的而不是以编程方式将布局中的UI组件绑定到应用程序中的数据源。免去了编写findViewById这样的模板代码，提升了程序性能，防止了内存泄漏和NPE等发生。

看一段ViewModel类的注释，ViewModel的思想不言而喻：
```
The purpose of the ViewModel is to acquire and keep the information that is necessary for an
Activity or a Fragment. The Activity or the Fragment should be able to observe changes in the
ViewModel. ViewModels usually expose this information via {@link LiveData} or Android Data
Binding. You can also use any observability construct from you favorite framework.
```

## 2 实践
### 2.1 依赖
DataBinding位于`com.android.databinding.xxx`模块下，在module的build.gradle里添加如下代码即可引入依赖：
```gradle
android {
    dataBinding {
        enabled = true
    }
    ...
}
```
![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/android_advance/1.png?raw=true)

### 2.2 绑定表达式
DataBinding的绑定代码写在layout文件里，用声明式的写法代替了以前代码主动find。

和以前写layout不同的是，要绑定数据的layout文件的根布局必须以`<layout></layout>`标签包裹。下面将一个用户信息类User和多个TextView进行绑定。

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <variable
            name="user"
            type="com.example.test.databinding.User" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.name}" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.phoneNum}" />

    </LinearLayout>
</layout>
```

定义了以User类，里面有姓名name和电话号码phoneNum两个成员变量。在`<layout>`标签下添加`<data>`标签，里面加入布局中需要用到的数据，每一个数据item用`<variable>`标签包裹，name是数据的名称，type则是类的绝对路径包名。TextView想使用user的变量就可以直接通过调用`@{user.xxx}`来获取。

    注意，User类必须要提供开放的getter方法共layout调用，等效成将变量设置成public。

表达式支持多种特性，包括运算符，方法调用，import包等，这里只是为了后续源码解析介绍简单操作，想要详细了解可以参考官方文档[data-binding/expressions](https://developer.android.com/topic/libraries/data-binding/expressions)。

### 2.3 绑定数据
```java
public class DataBindingActivity extends AppCompatActivity {
	@Override
	protected void onCreate(@Nullable Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		ActivityDatabindingBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_databinding);
		User user = new User("kinsomy", "123121");
		binding.setUser(user);
	}
}
```

在Activity的onCreate中，不再调用setContentView(),取而代之调用DataBindingUtil内的setContentView()方法，传入当前activity和layout资源id。`ActivityDatabindingBinding`是在layout文件写完之后重新build项目自动生成的类，它的命名规则是layout名字的大驼峰+Binding结尾。然后再用setUser()方法将user实例塞到binding实例里，这样user数据就渲染到界面中了。这里的源码下文会详细解析。

### 2.4 事件绑定
DataBinding中的事件绑定有多种写法：
#### 直接设置
```xml
<Button
    android:id="@+id/btn"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="normal onClick" />
```
```java
binding.btn.setOnClickListener(new View.OnClickListener() {
	@Override
	public void onClick(View v) {
		user.setName("normal onclick");
		user.setPhoneNum("normal onclick");
		binding.setUser(user);
	}
});
```

#### 方法引用
```xml
<variable
    name="listener"
    type="com.example.test.databinding.OnBindingClickListener"/>

<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:onClick="@{listener::onMethodReferenceClick}"
    android:text="onMethodReferenceClick" />
```

```java
public interface OnBindingClickListener {

	void onMethodReferenceClick(View view);
}
binding.setListener(new OnBindingClickListener() {
	@Override
	public void onMethodReferenceClick(View view) {
		user.setName("onMethodReferenceClick");
		user.setPhoneNum("onMethodReferenceClick");
		binding.setUser(user);
	}
});
```

方法引用和原来的在layout内写onclick时间类似。与View onClick属性相比，一个主要优点是表达式在编译时处理，因此如果该方法不存在或其签名不正确，则会收到编译时错误。

#### 监听器绑定
```xml
<variable
    name="listener"
    type="com.example.test.databinding.OnBindingClickListener"/>

<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:onClick="@{() -> listener.onListenerBindingClick()}"
    android:text="onListenerBindingClick" />
```

```java
public interface OnBindingClickListener {
	void onListenerBindingClick();
}
binding.setListener(new OnBindingClickListener() {

	@Override
	public void onListenerBindingClick() {
		user.setName("onListenerBindingClick");
		user.setPhoneNum("onListenerBindingClick");
		binding.setUser(user);
	}
});
```
监听器绑定需要在处理事件处传入lambda表达式，可以传入任意数据格式，只要和规定的方法签名相同即可。这种写法只在事件发生时才会开始处理。

监听器绑定和方法引用最大的区别在于方法引用实在编译时就创建的，但是监听器绑定是在事件触发时绑定的，可以实时决定处理事件的方法。

### 2.5 双向绑定
上文只做到了通过binding实例修改user对象属性去改变UI，这只是单向绑定，要做到双向绑定还要能通过修改UI去改变user实例。双向绑定的写法如下：
```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@{user.name}" />

<EditText
    android:layout_width="200dp"
    android:layout_height="wrap_content"
    android:hint="修改名字"
    android:text="@={user.name}" />
```

输入框EditText用了`@={user.name}`标记，比普通的赋值标记多了`"="`,接受name变量的赋值同时监听用户对name的更新。

然后去需要修改User类，让他继承BaseObservable，并在getter函数上添加`@Bindable`注解，这会编译时生成一个BR类，里面记录着所有被`@Bindable`注解标的field。在setter方法里调用`notifyPropertyChanged(int fieldId)`传入BR类里的fieldId。

生成的BR类：
```java
public class BR {
  public static final int _all = 0;

  public static final int phone = 1;

  public static final int name = 2;

  public static final int listener = 3;
}
```
```java
public class User extends BaseObservable {
	private String name;
	private String phone;

	public User(String name, String phone) {
		this.name = name;
		this.phone = phone;
	}

	@Bindable
	public String getName() {
		return name;
	}

	public void setName(String name) {
		if (this.name != name) {
			this.name = name;
			notifyPropertyChanged(BR.name);
		}
	}

	@Bindable
	public String getPhone() {
		return phone;
	}

	public void setPhone(String phone) {
		if (this.phone != phone) {
			this.phone = phone;
			notifyPropertyChanged(BR.phone);
		}
	}
}
```

做了上面的改动之后，不管edittext输入上面，textview都会随之更新。

## 3 源码分析
在写完layout文件之后，build项目会得到以layout文件名大驼峰+”Binding“结尾的生成抽象类`ActivityDataBindingBinding`和实现类`ActivityDatabindingBindingImpl`。
在上面ActivityDataBindingBinding实例是通过`DataBindingUtil.bind`创建出来的，所以先从这个类着手看。

### 3.1 DataBindingUtil
DataBindingUtil的作用就是用来创建binding对象,提供了多种静态方法：
```java
public static <T extends ViewDataBinding> T inflate(@NonNull LayoutInflater inflater,
            int layoutId, @Nullable ViewGroup parent, boolean attachToParent) 

public static <T extends ViewDataBinding> T bind(@NonNull View root)

public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
        int layoutId) {
    return setContentView(activity, layoutId, sDefaultComponent);
}

public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
            int layoutId, @Nullable DataBindingComponent bindingComponent) {
        activity.setContentView(layoutId);
        View decorView = activity.getWindow().getDecorView();
        ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
        return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
    }
        ...
```
详细看一下setContentView的创建方式，参数传入activity上下文，layout文件id，第一步就会调用activity的setContentView方法，所以可以放心取代以前的setContentView，然后获得当前activity的DecorView实例，通过DecorView拿到根布局contentView，然后调用bindToAddedViews,这个方法会找到当前布局下的所有子View，然后依次对其调用bind方法。
```java
private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
            ViewGroup parent, int startChildren, int layoutId) {
        final int endChildren = parent.getChildCount();
        final int childrenAdded = endChildren - startChildren;
        if (childrenAdded == 1) {
            final View childView = parent.getChildAt(endChildren - 1);
            return bind(component, childView, layoutId);
        } else {
            final View[] children = new View[childrenAdded];
            for (int i = 0; i < childrenAdded; i++) {
                children[i] = parent.getChildAt(i + startChildren);
            }
            return bind(component, children, layoutId);
        }
    }

static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View root,
        int layoutId) {
    return (T) sMapper.getDataBinder(bindingComponent, root, layoutId);
}
```
bind方法会调用DataBinderMapperImpl的getDataBinder方法，追进去看：
```java
public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
  int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
  if(localizedLayoutId > 0) {
    final Object tag = view.getTag();
    if(tag == null) {
      throw new RuntimeException("view must have a tag");
    }
    switch(localizedLayoutId) {
      case  LAYOUT_ACTIVITYDATABINDING: {
        if ("layout/activity_databinding_0".equals(tag)) {
          return new ActivityDatabindingBindingImpl(component, view);
        }
        throw new IllegalArgumentException("The tag for activity_databinding is invalid. Received: " + tag);
      }
    }
  }
  return null;
}
```
这也是一个生成类，里面实际是调用了ViewDataBinding的子类，也就是生成的ActivityDataBindingBindingImpl来构造出ActivityDataBindingBinding实例返回的。下面就来看看ActivityDataBindingBindingImpl做了些什么。

### 3.2 ActivityDataBindingBindingImpl
先看上面用到的构造函数
```java
public ActivityDatabindingBindingImpl(@Nullable android.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
    this(bindingComponent, root, mapBindings(bindingComponent, root, 8, sIncludes, sViewsWithIds));

private ActivityDatabindingBindingImpl(android.databinding.DataBindingComponent bindingComponent, View root, Object[] bindings) {
    super(bindingComponent, root, 1
        , (android.widget.Button) bindings[7]
        );
    this.mboundView0 = (android.widget.LinearLayout) bindings[0];
    this.mboundView0.setTag(null);
    this.mboundView1 = (android.widget.TextView) bindings[1];
    this.mboundView1.setTag(null);
    this.mboundView2 = (android.widget.TextView) bindings[2];
    this.mboundView2.setTag(null);
    this.mboundView3 = (android.widget.EditText) bindings[3];
    this.mboundView3.setTag(null);
    this.mboundView4 = (android.widget.EditText) bindings[4];
    this.mboundView4.setTag(null);
    this.mboundView5 = (android.widget.Button) bindings[5];
    this.mboundView5.setTag(null);
    this.mboundView6 = (android.widget.Button) bindings[6];
    this.mboundView6.setTag(null);
    setRootTag(root);
    // listeners
    mCallback1 = new com.example.test.generated.callback.OnClickListener(this, 1);
    invalidateAll();
```
通过mapBindings方法遍历view的层层级找到所有的绑定的和有Id的view，返回一个数组，有id的view在抽象类ActivityDatabindingBinding中被复制，其余的绑定数据的view被命名为mboundView+数字序列。

再看mapBindings方法，解析在注释里：
```java
private static void mapBindings(DataBindingComponent bindingComponent, View view,
        Object[] bindings, IncludedLayouts includes, SparseIntArray viewsWithIds,
        boolean isRoot) {
    final int indexInIncludes;
    final ViewDataBinding existingBinding = getBinding(view);
    if (existingBinding != null) {
        return;
    }
    Object objTag = view.getTag();
    final String tag = (objTag instanceof String) ? (String) objTag : null;
    boolean isBound = false;
    //如果是根布局layout
    if (isRoot && tag != null && tag.startsWith("layout")) {
        final int underscoreIndex = tag.lastIndexOf('_');
        if (underscoreIndex > 0 && isNumeric(tag, underscoreIndex + 1)) {
            //找到 ”binding_num“中的num值
            final int index = parseTagInt(tag, underscoreIndex + 1);
            if (bindings[index] == null) {
                //将找到的根布局添加到数组
                bindings[index] = view;
            }
            //判断是否有include布局，并更新include布局在layout层级的index
            indexInIncludes = includes == null ? -1 : index;
            isBound = true;
        } else {
            indexInIncludes = -1;
        }
        //如果已经绑定（binding_ 开头）
    } else if (tag != null && tag.startsWith(BINDING_TAG_PREFIX)) {
        int tagIndex = parseTagInt(tag, BINDING_NUMBER_START);
        if (bindings[tagIndex] == null) {
            bindings[tagIndex] = view;
        }
        isBound = true;
        indexInIncludes = includes == null ? -1 : tagIndex;
    } else {
        // Not a bound view
        indexInIncludes = -1;
    }
    //如果没有绑定，就找有id声明的
    if (!isBound) {
        final int id = view.getId();
        if (id > 0) {
            int index;
            if (viewsWithIds != null && (index = viewsWithIds.get(id, -1)) >= 0 &&
                    bindings[index] == null) {
                bindings[index] = view;
            }
        }
    }
    //如果传入的viewgroup 递归遍历
    if (view instanceof  ViewGroup) {
        final ViewGroup viewGroup = (ViewGroup) view;
        final int count = viewGroup.getChildCount();
        int minInclude = 0;
        for (int i = 0; i < count; i++) {
            final View child = viewGroup.getChildAt(i);
            boolean isInclude = false;
            if (indexInIncludes >= 0 && child.getTag() instanceof String) {
                String childTag = (String) child.getTag();
                if (childTag.endsWith("_0") &&
                        childTag.startsWith("layout") && childTag.indexOf('/') > 0) {
                    // This *could* be an include. Test against the expected includes.
                    int includeIndex = findIncludeIndex(childTag, minInclude,
                            includes, indexInIncludes);
                    if (includeIndex >= 0) {
                        isInclude = true;
                        minInclude = includeIndex + 1;
                        final int index = includes.indexes[indexInIncludes][includeIndex];
                        final int layoutId = includes.layoutIds[indexInIncludes][includeIndex];
                        int lastMatchingIndex = findLastMatching(viewGroup, i);
                        if (lastMatchingIndex == i) {
                            bindings[index] = DataBindingUtil.bind(bindingComponent, child,
                                    layoutId);
                        } else {
                            final int includeCount =  lastMatchingIndex - i + 1;
                            final View[] included = new View[includeCount];
                            for (int j = 0; j < includeCount; j++) {
                                included[j] = viewGroup.getChildAt(i + j);
                            }
                            bindings[index] = DataBindingUtil.bind(bindingComponent, included,
                                    layoutId);
                            i += includeCount - 1;
                        }
                    }
                }
            }
            if (!isInclude) {
                mapBindings(bindingComponent, child, bindings, includes, viewsWithIds, false);
            }
        }
    }
}
```

这样就初始化了所有绑定的view的实例。
