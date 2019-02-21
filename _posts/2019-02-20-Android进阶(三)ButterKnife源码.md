---
layout:     post                    # 使用的布局（不需要改）
title:   Android进阶(三)ButterKnife源码解析          # 标题 
subtitle:   Android advance #副标题
date:       2019-02-20            # 时间
author:     Kinsomy                      # 作者
header-img:    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
---

## 1 概述
ButterKnife是Android用于视图绑定的依赖注入框架，用注解来生成模板代码。

通过分析ButterKnife源码可以加深对注解使用以及依赖注入概念的理解。

## 2 ButterKnife使用
### 2.1 引入依赖
在Project的`build.gradle`文件中添加依赖：
```gradle
dependencies {
  implementation 'com.jakewharton:butterknife:10.1.0'
  annotationProcessor 'com.jakewharton:butterknife-compiler:10.1.0'
}
```

### 2.2 绑定视图控件
```java
class ExampleActivity extends Activity {
  @BindView(R.id.tv_name) EditText username;
  @BindView(R.id.tv_pwd) EditText password;
  @Override 
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```
绑定视图控件需要在`onCreate()`的setContentView之后加上bind代码，然后使用`@BindView`注解绑定资源到成员变量。免去了findViewById的操作。

### 2.3 绑定资源
```java
@BindString(R.string.login_error) String loginErrorMessage;
@BindColor(R.color.color_error_text) int colorErrorText;
```

### 2.4 绑定点击事件
```java
@OnClick(R.id.btn_login)
    void onLogin() {
        //处理登录
    }
```

这里简单讲了常用的注解，ButterKnife所有的注解全部在[butterknife-annotations](https://github.com/JakeWharton/butterknife/tree/master/butterknife-annotations) library内。

## 3 @Retention
`@Retention`这个注解是声明注解的保留策略，有三种类型。

* @Retention(SOURCE) 源码级别的注解,注解只会在java文件中保留,源码被编译成class文件后,注解信息就会消失。

* @Retention(CLASS) 编译时注解,注解在java文件中保留，被译成class文件同样被保留,在JVM运行程序时会丢弃注解信息。

* @Retention(RUNTIME) 运行时注解，JVM运行程序会保留注解信息,需要通过反射获取注解信息。

ButterKnife在9.0之前一直是用的编译时注解，9.0开始转为使用新的运行时注解。
## 4 ButterKnife8.x版本分析
### 4.1 自定义注解
```java
@Retention(CLASS) @Target(FIELD)
public @interface BindView {
  /** View ID to which the field will be bound. */
  @IdRes int value();
}
```
可以看到注解BindView的保留策略是CLASS级别，注解范围FIELD，表示只能标记变量。参数用方法`int value()`表示，只能传入资源id。

### 4.2 bind方法
每次想要使用ButterKnife绑定控件的时候都需要先调用`ButterKnife.bind();`方法，来看下bind的源码。
```java
public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);
}
public static Unbinder bind(@NonNull View target) {
    return createBinding(target, target);
}
public static Unbinder bind(@NonNull Dialog target) {
    View sourceView = target.getWindow().getDecorView();
    return createBinding(target, sourceView);

}
...
```
可以看到bind有多个同名方法，参数target可以穿Activity，view，dialog等，这样就可以在多种情况下使用ButterKnife。方法全部返回Unbinder实例。想要接触view绑定，则可以调用Unbinder的unbind方法。

看第一个bind方法传入Activity的时候，会先拿到Activity的DecorView，然后传入`createBinding`绑定Activity和view。

```java
static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } 
    ....
}

private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
}
```
在`createBinding`中首先会调用`findBindingConstructorForClass`获得一个`Constructor<? extends Unbinder>`实例，看一下获取过程，先从BINDINGS的Map中去找是否已经存在target类的实例，如果有缓存就直接返回，没有则通过反射去加载`clsName + "_ViewBinding"`这个类，这里先记住`_ViewBinding`结尾的类，后面会讲到，这是一个使用apt自动生成的类，获取到实例后将它加入BINDINGS缓存，随即返回。然后继续回到`createBinding`方法，拿到`Constructor`实例就调用`newInstance`构造方法构造一个Unbinder。

### 4.3 注解处理器
编译时注解都会用到注解处理器，注解处理器回去找到自定义的注解进行处理，需要继承抽象类`AbstractProcessor`，重写它的方法。ButterKnife的注解处理器叫`ButterKnifeProcessor`。位于[butterknife-compiler](https://github.com/JakeWharton/butterknife/tree/master/butterknife-compiler)模块下。
```java
//AutoService注解 自动生成Processor文件
@AutoService(Processor.class)
public final class ButterKnifeProcessor extends AbstractProcessor {
	@Override
	public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
		Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

		for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
			TypeElement typeElement = entry.getKey();
			BindingSet binding = entry.getValue();

			JavaFile javaFile = binding.brewJava(sdk, debuggable);
			try {
				javaFile.writeTo(filer);
			} catch (IOException e) {
				error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
			}
		}

		return false;
	}
	private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
		Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
		Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();
		scanForRClasses(env);

        ...
        
		// Process each @BindView element.
		for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
			// we don't SuperficialValidation.validateElement(element)
			// so that an unresolved View type can be generated by later processing rounds
			try {
				parseBindView(element, builderMap, erasedTargetNames);
			} catch (Exception e) {
				logParsingError(element, BindView.class, e);
			}
		}

		// Associate superclass binders with their subclass binders. This is a queue-based tree walk
		// which starts at the roots (superclasses) and walks to the leafs (subclasses).
		Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
				new ArrayDeque<>(builderMap.entrySet());
		Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
		while (!entries.isEmpty()) {
			Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

			TypeElement type = entry.getKey();
			BindingSet.Builder builder = entry.getValue();

			TypeElement parentType = findParentType(type, erasedTargetNames);
			if (parentType == null) {
				bindingMap.put(type, builder.build());
			} else {
				BindingSet parentBinding = bindingMap.get(parentType);
				if (parentBinding != null) {
					builder.setParent(parentBinding);
					bindingMap.put(type, builder.build());
				} else {
					// Has a superclass binding but we haven't built it yet. Re-enqueue for later.
					entries.addLast(entry);
				}
			}
		}

		return bindingMap;
	}
}
```
注解处理器中最主要的方法就是process，用来对各个自定义注解做处理，方法第一行调用`findAndParseTargets`方法找到所有的注解。调用`env.getElementsAnnotatedWith(BindView.class)`这个方法去找到环境中所有用到bindview注解的地方，然后依次遍历，调用`parseBindView`方法，该方法内首先做一些正确性校验，然后再看buildMap缓存里是否已经解析过该注解，如果已经解析过则直接返回，否则调用`getOrCreateBindingBuilder`生成`BindingSet.Builder`实例对象并且加入到builderMap缓存中去。newBuilder方法会生成一个builder实例，在这里我们看到了ClassName.get(packageName, className + "_ViewBinding")这一行代码，他就是上文所看到的以viewbinding结尾的自动生成文件，这个文件是在注解处理器的`process`方法里调用`binding.brewJava`生成的。


```java 
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
							   Set<TypeElement> erasedTargetNames) {
		TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

		// Assemble information on the field.
		int id = element.getAnnotation(BindView.class).value();

		BindingSet.Builder builder = builderMap.get(enclosingElement);
		QualifiedId qualifiedId = elementToQualifiedId(element, id);
		if (builder != null) {
			String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));
			if (existingBindingName != null) {
				error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
						BindView.class.getSimpleName(), id, existingBindingName,
						enclosingElement.getQualifiedName(), element.getSimpleName());
				return;
			}
		} else {
			builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
		}
}
private BindingSet.Builder getOrCreateBindingBuilder(
	Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
	BindingSet.Builder builder = builderMap.get(enclosingElement);
	if (builder == null) {
	    builder = BindingSet.newBuilder(enclosingElement);
		builderMap.put(enclosingElement, builder);
	}
	return builder;
}

  static Builder newBuilder(TypeElement enclosingElement) {
    String packageName = getPackage(enclosingElement).getQualifiedName().toString();
    String className = enclosingElement.getQualifiedName().toString().substring(
        packageName.length() + 1).replace('.', '$');
    ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");

    boolean isFinal = enclosingElement.getModifiers().contains(Modifier.FINAL);
    return new Builder(targetType, bindingClassName, isFinal, isView, isActivity, isDialog);
  }
```

### 4.4 _ViewBinding文件
接下来看看自动生成的_ViewBinding文件里有什么内容。
```java
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.mTextView = Utils.findRequiredViewAsType(source, R.id.text_view, "field 'mTextView'", TextView.class);
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    target.mTextView = null;
  }
}

public static View findRequiredView(View source, @IdRes int id, String who) {
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
    String name = getResourceEntryName(source, id);
    throw new IllegalStateException("Required view '"
        + name
        + "' with ID "
        + id
        + " for "
        + who
        + " was not found. If this view is optional add '@Nullable' (fields) or '@Optional'"
        + " (methods) annotation.");
}
```
这里有两个构造方法，其中MainActivity_ViewBinding(MainActivity target, View source)就对应于上文用反射来构造Unbinder实例的`constructor.newInstance`方法,而source就是传进来的DecorView，通过`findRequiredViewAsType`去找到textview控件并复制给Activity的成员变量`mTextView`。这样通过ButterKnife就可以把xml里的控件和变量绑定起来了，`findRequiredViewAsType`最后实际就是调用了`findViewById`。