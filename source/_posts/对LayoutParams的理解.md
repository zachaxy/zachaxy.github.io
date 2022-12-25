---
title: 对LayoutParams的理解
date: 2017-05-11 09:33:25
tags: Android Framework
---

# LayoutParams 是什么？

相信很多人都有疑惑，LayoutParams 是什么？不知道大家想过没有，如果定义一个 ViewGroup，然后在这个 ViewGroup 中放若干 view，这些 view 在 ViewGroup 中的大小关系是保存在哪里的呢？

```xml
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.zx.readsourcecode.MainActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="文字"/>

</LinearLayout>
```

上面是一段最简单不过的布局文件，这里我们最常用的一个属性`android:layout_width`和`android:layout_height`相信大家再熟悉不过，其实这两个属性对应两个变量 width 和 height，那么这两个变量是哪个类的成员变量呢？答案就是 LayoutParams。虽然之前不知道 LayoutParams 这和类的存在，但是却一直偷偷的使用它。



# LayoutParams 的作用

 **LayoutParams 是 View 用来告诉它的父控件如何放置自己的。**基类 LayoutParams（也就是 ViewGroup.LayoutParams）仅仅描述了这个 View 想要的宽度和高度。在 View 的测量过程中，LayoutParams 起到了决定性的作用！



注意：`android:layout_width`和`android:layout_height`表面看起来是 TextView 的宽度和高度，相信这是很多人误解的地方，但如果是描述自己的宽高，那么属性名直接定义为`android:width`和`android:height`不就行了，为什么前面要加`layout_`的前缀？这是因为 TextView 既然放在 LinearLayout 中，那么它的宽度和高度并不是由自己决定的，而是由其父容器（这里对应 LinearLayout）来决定的。通过该属性可选的值 `match_parent`可以看出，这并不是一个具体的值，而是描述为和父容器一样宽，所以宽度是看父容器有多宽，才给子 view 设置多宽的。



而描述 View 直接用它们自己的属性就好了，如`textColor`和`text`等等，为什么还需要引入 LayoutParams 呢？因为`textColor`和`text`等这样的属性都是只与 TextView 自身有关的，无论这个 TextView 处于什么环境，这些属性都是不变的。而`layout_width`与`layout_marginLeft`这样的属性是与它的父控件息息相关的，是父控件通过 LayoutParams 提供这些”layout_”属性给孩子们用的；是父控件根据孩子们的要求（LayoutParams）来决定怎么测量。



所以在 xml 布局中，凡是以`layout_`开头的属性，均不是 view 自身的，而是它所在的父控件的 LayoutParams 类中的属性，这里的父控件一定是 ViewGroup 的一个子类。



# ViewGroup.LayoutParams

LayoutParams 是 ViewGroup 中的一个静态类，而每个 ViewGroup 的子类（eg：LinearLayout，FrameLayout，RelativeLayout 等）中依然有一个各自的 LayoutParams 的子类，其在 ViewGroup 的 LayoutParams 静态类的基础之上，添加了和具体布局相关的属性，比如 LinearLayout 中特有的 layout_weight 属性，用来指定特定方向的分配比例，再比如 RelativeLayout 特有的 layout_left，用来指定当前 view 和其它子 view 相对位置的属性等。进入源码中看一下 ViewGroup 中静态内部类 LayoutParams。

```
public static class LayoutParams {

    @SuppressWarnings({"UnusedDeclaration"})
    @Deprecated
    public static final int FILL_PARENT = -1;

    public static final int MATCH_PARENT = -1;

    public static final int WRAP_CONTENT = -2;

    public int width;

    public int height;
    
    public LayoutParams(Context c, AttributeSet attrs) {
    TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
    
    	setBaseAttributes(a,
   			 R.styleable.ViewGroup_Layout_layout_width,
   		 	R.styleable.ViewGroup_Layout_layout_height);
    
    	a.recycle();
    }

    public LayoutParams(int width, int height) {
        this.width = width;
        this.height = height;
    }
    ......
}
```

可以看到其内部的成员变量只有最基本的 width 和 height，对应了布局文件中的`android:layout_width`和`android:layout_height`的值，在解析布局文件时，遇到这两个属性，就给对应的 LayoutParams 的这两个成员变量进行赋值了。



在 ViewGroup 中还自带了一个 LayoutParams 的子类 MarginLayoutParams，供不同的布局使用

```java
public static class MarginLayoutParams extends ViewGroup.LayoutParams {

    public int leftMargin;

    public int topMargin;

    public int rightMargin;

    public int bottomMargin;

    private int startMargin = DEFAULT_MARGIN_RELATIVE;

    private int endMargin = DEFAULT_MARGIN_RELATIVE;

    ...
} 
```

其添加了 Margin 相关的属性，相对于布局文件的`android:layout_marginLeft="5dp"`等属性。



`LayoutParams`其实是父控件提供给子 view 的，好让子 view 选择如何测量和放置自己。所以在子 view 添加到父控件的那一刻，子 view 就应该有`LayoutParams`了。我们来看看几中常见的在代码中添加 View 的方式：还是上面的布局，这次我们不用 xml，而是用代码来实现一样的效果。 

```java
LinearLayout linearLayout = (LinearLayout) findViewById(R.id.activity_main);
TextView textView = new TextView(this);
LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(
        LinearLayout.LayoutParams.MATCH_PARENT,
        LinearLayout.LayoutParams.WRAP_CONTENT);
linearLayout.addView(textView, params);
```



View 要是想被添加到 ViewGroup 中，是不可能脱离相应的 ViewGroup.LayoutParams，而独立存在的，不信你将布局文件中的`android:layout_width`和`android:layout_height`这两个属性删掉，编译器立即报错。

同时在代码中添加上述操作，虽然可以使用如下代码：

```java
LinearLayout linearLayout = (LinearLayout) findViewById(R.id.activity_main);
TextView textView = new TextView(this);
linearLayout.addView(textView);
```

但是我们追到 addView 方法内部看一下，可以得出结论，调用 addView（view）的方法，对应的 ViewGroup子类会通过 generateDefaultLayoutParams（）方法自动添加一个默认的 LayoutParams。不同的 ViewGroup（这里指的是 LinearLayout）产生具体 LayoutParams 的方法不同。

```java
public void addView(View child) {
    addView(child, -1);
}


public void addView(View child, int index) {
    if (child == null) {
    	throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
    	params = generateDefaultLayoutParams();
    if (params == null) {
    	throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
    	}
    }
    addView(child, index, params);
}
```



不同的 ViewGroup（布局）产生具体 LayoutParams 的方法不同。看一下 LinearLayout 中的 generateDefaultLayoutParams（）是如何添加的：

```java
protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```

很简单的添加了一个宽高均为 wrap_content 的属性。

**注：generateDefaultLayoutParams 方法中返回的 LayoutParams 类，是具体的 ViewGroup 子类（这里代表 LinearLayout）中的静态类 LayoutParams，而不是 ViewGroup 中的。**



在深入到 addView 中，看一下 view 和 LayoutParams 是如何关联的，这里删除了部分代码，只看主线；

```java
private void addViewInner(View child, int index, LayoutParams params,
        boolean preventRequestLayout) {  

	//(1)
    if (child.getParent() != null) {
        throw new IllegalStateException("The specified child already has a parent. " +
                "You must call removeView() on the child's parent first.");
    }

  //(2)
    if (!checkLayoutParams(params)) {
        params = generateLayoutParams(params);
    }

  	//(3)preventRequestLayout 参数的默认值是 false；
    if (preventRequestLayout) {
        child.mLayoutParams = params;
    } else {
        child.setLayoutParams(params);
    }

  	//(4)默认值为 -1
    if (index < 0) {
        index = mChildrenCount;
    }

  //(5)
    addInArray(child, index);

    //(6) tell our children，默认值为 false；
    if (preventRequestLayout) {
        child.assignParent(this);
    } else {
        child.mParent = this;
    }

  //(7)
    dispatchViewAdded(child);
 
  //......
 
}
```

这里看一下 addViewInner 方法内部的执行流程

1. 检查 view 是否被别的 ViewGroup 添加过，如果被别的添加过，那么抛出异常
2. 再对 layoutParams 进行检测，如果为 null，执行 generateLayoutParams(params)，这也是最后一次为 view 创建 params 了，这里添加的是 ViewGroup.LayoutParams，一般在 LinearLayout、RelativeLayout 等已有的布局中，addview 中会先执行类似的为 view 创建 params 的方法，添加的是各布局特有的 LayoutParams，但是如果是我们自定义的 ViewGroup，也许没有实现配套的 LayoutParams，所以最后用的是 ViewGroup 中自带的 LayoutParams
3. 将 view 的 layoutParams 设置为当前的 layoutParams，在这里执行了关联
4. 除非我们在 addView(view,index)指定了具体值，否则 index 默认值为 -1，如果是 -1 的话，这一步将 index 改为 ViewGroup 中当前 child view 的总和
5. ViewGroup 中有一个数组，保存着其所有子 view，这里根据 index 值添加到相应的位置，当然这里也涉及到了数组扩容问题，如果你手动指定的 index 过大，超过了当前 ViewGroup 的 mChildrenCount，那么直接报错，index 过大
6. 为 view 分配 parent，这个 parent 就是添加该子 view 的 ViewGroup
7. 执行了 addView 方法，那么原有的 ViewGroup 的结构势必会发生改变，这时候会执行一个回调，通知当前 ViewGroup 添加或者移除了一个 view；当然这个回调默认是 null，我们可以手动添加该回调，使用`setOnHierarchyChangeListener(OnHierarchyChangeListener listener)`





# 自定义 LayoutParams

前面提到，每个 ViewGroup 的子类（eg：LinearLayout，FrameLayout，RelativeLayout 等）中依然有一个各自的静态内部类 xxxLayout.LayoutParams，其继承了 ViewGroup#LayoutParams，并根据具体的布局，新增了相关属性，比如 LinearLayout 中特有的 layout_weight 属性，用来指定特定方向的分配比例，再比如 RelativeLayout 特有的 layout_left、layout_right 等，用来指定当前 view 和其它子 view 相对位置的属性等。

前面分析 addView 的流程中也介绍到子 view 要想添加到父容器中，父容器必定会给子 view 指定一个 LayoutParams，脱离了 LayoutParams，view 无法单独存在，view 可以通过其 getLayoutParams 方法获取其父容器为子 view 绑定的 LayoutParams。



使用 Android 官方提供的这些 ViewGroup（eg：LinearLayout，FrameLayout，RelativeLayout 等）在 addView 时都会为 view 设置一个 xxxLayout.LayoutParams，而在我们自定义 ViewGroup 控件的时候，如果没有为自定义的 ViewGroup 添加配套的 LayoutParams，那么最终在执行 addViewInner 方法时，会为 view 添加 ViewGroup#LayoutParams，这个 LayoutParams 是一个最基础的，只有 width 和 height 属性，如果我们想实现自定义的 ViewGroup 特殊需求，那么自定义 LayoutParams 是必不可少的。



关于如何自定义 LayoutParams，以及在自定义 LayoutParams 时要注意哪些问题，请参考实战案例[自定义 ViewGroup——百分比布局]()，这篇文章会根据具体的 ViewGroup 的功能需求，制定对应的 LayoutParams。



# 总结

LayoutParams 的作用：是 View 用来告诉它的父控件如何放置自己的，理解这一点很重要，因为这是 view 在 measure 和 layout 过程中很重要的因素。