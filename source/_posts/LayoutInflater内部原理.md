---
title: LayoutInflater内部原理
date: 2017-05-12 22:36:24
tags: Android Framework
---

# LayoutInflater

相信大家对 LayoutInflater 并不陌生，我们在使用 ListView，RecyclerView 时，经常需要将 item 的布局文件加载到 ListView 上，这就用到了 LayoutInflater，调用 inflate 方法就可以将一项 item 添加到父控件中，那么它内部是如何实现的呢？



先来看一下我们平时是怎么获取 LayoutInflater 的

- LayoutInflater layoutInflater = LayoutInflater.from(context);
- LayoutInflater layoutInflater = (LayoutInflater) context  .getSystemService(Context.LAYOUT_INFLATER_SERVICE); 

其实第一种内部是调用了第二种方法，这是一个跨进程通信的过程，因为解析布局文件这个服务是系统级的服务。



在获取到 LayoutInflater 之后，就可以使用其方法了,其对外提供的 inflate 方法有若干重载方法，但是最后都是调用到了下面的方法：

```java
inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) 
```

接下来进入 inflate 方法内部，看其如何实现：

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

在第 10 行，根据传入的布局文件 id，生成一个 Xml 解析器，LayoutInflater 其实就是使用 Android 提供的 pull 解析方式来解析布局文件的。接下来调用 inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)方法开始解析所提供的布局文件了

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;

        try {
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

            final String name = parser.getName();
            
            if (DEBUG) {
                System.out.println("**************************");
                System.out.println("Creating root view: "
                        + name);
                System.out.println("**************************");
            }

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                if (DEBUG) {
                    System.out.println("-----> start inflating children");
                }

                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);

                if (DEBUG) {
                    System.out.println("-----> done inflating children");
                }

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        } catch (XmlPullParserException e) {
            InflateException ex = new InflateException(e.getMessage());
            ex.initCause(e);
            throw ex;
        } catch (Exception e) {
            InflateException ex = new InflateException(
                    parser.getPositionDescription()
                            + ": " + e.getMessage());
            ex.initCause(e);
            throw ex;
        } finally {
            // Don't retain static reference on context.
            mConstructorArgs[0] = lastContext;
            mConstructorArgs[1] = null;
        }

        Trace.traceEnd(Trace.TRACE_TAG_VIEW);

        return result;
    }
}
```

第 6 行，创建一个 AttributeSet 对象，该对象用来保存 xml 布局中 View 中设置的属性，只不过现在只是创建了这个类，对于该类的成员变量现在还没有进行初始化，其实这个类中持有了 xmlParser(这一点要明确)，其内部封装了 getAttributeByName 等方法，并且在接下来的解析过程中从来没有显示的解析过 view 的属性，而是在反射创建 view 的过程中，传入这个 AttributeSet 对象，想要获取什么属性，直接调用 AttributeSet 的 getAttributeByXXX 的方法即可。至于和 layout 相关的属性，都不在 view 的构造方法中获取，而是在为其设置 LayoutParams 的时候，进行获取。

```java
public LayoutParams(Context c, AttributeSet attrs) {
    TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.ViewGroup_Layout);
    setBaseAttributes(a,
            R.styleable.ViewGroup_Layout_layout_width,
            R.styleable.ViewGroup_Layout_layout_height);
    a.recycle();
}
```
第 9 行，将传入的第二个参数 root 赋给 result，先不管这个方法内部做了什么，总之这个方法返回值就是 result，但是也不一定就是第二个参数 root，因为该方法中间还有可能改变 result 的值。

第 14 行，开始解析 xml 布局文件，这里是个 while 循环，循环体中是空实现，目的是要获得该 xml 的开始节点或者结尾节点。如果不是开始或者结束，那么就一直寻找 Dom 树中的下一个节点，直到找到为止。当然这里还是为了获取开始节点，因为在第 19 行还会判断找到的是不是开始节点，不是的话直接报错了。

第 24 行，获取到传入的布局文件的根节点的 name

第 33 行，如果该节点是"merge"，会进行一次判断，因为 merge 的用法是会将 merge 节点下的所有布局全部添加到其父容器中，如果我们传入的 root 为 null，或者 attachToRoot 为 false，违背了 merge 的用法，直接报错；否则会调用 rInflate 方法解析 merge 节点中的控件，rInflate 的内部实现在下面进行分析

第 42 行，如果根节点不是"merge"，那么调用`createViewFromTag(root, name, inflaterContext, attrs)`方法，创建该布局根节点所代表的 View 对象 temp，createViewFromTag 方法的内部流程在下面进行解释。

第 46 行，判断该 inflate 方法的第二个参数 root 是否为 null，如果不为空，那么调用该 root 的 generateLayoutParams 方法产生 LayoutParams

第 52 行，判断该 inflate 方法的第三个参数 attachToRoot 的值，如果为 false，那么给第 42 行返回的布局的根 View 设置第 46 行创建的 LayoutParams

第 65 行，执行 rInflateChildren(parser, temp, attrs, true)方法，解析 temp 节点内的子 view，内部执行流程在下面进行分析

第 73 行，如果同时满足 root 不为 null 并且 attachToRoot 为 true，那么执行 root.addView(temp, params)，并最终返回该 root（inflate 的第二个参数）

第 79 行，如果 root 为 null 或者 attachToRoot 为 false，那么直接返回 temp（解析的布局文件的根节点的 View 对象）



# LayoutInflater 方法使用不同参数产生不同的效果

现在有如下两个布局文件：

activity_main.xml ：

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
</LinearLayout>
```

button_layout.xml：

```xml
<Button xmlns:android="http://schemas.android.com/apk/res/android"  
    android:layout_width="wrap_content"  
    android:layout_height="300dp"  
    android:text="Button" >  
</Button> 
```

我们并没有在 LinearLayout 中添加 Button 的标签，而是将 Button 写在另一个单独的布局文件中，然后在代码中将 button 添加到上面的 LinearLayout 中,但是使用不同的 inflate 的参数，查看结果：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	getWindow().requestFeature(Window.FEATURE_NO_TITLE);
	setContentView(R.layout.activity_main);

	LinearLayout linearLayout = (LinearLayout) findViewById(R.id.activity_main);
	Button button;
	LayoutInflater inflater = LayoutInflater.from(this);
	//（1）
	button = (Button) inflater.inflate(R.layout.button_main, null);
	linearLayout.addView(button);

  	//（2）
	inflater.inflate(R.layout.button_main, linearLayout);
	
  	//（3）
	button = (Button) inflater.inflate(R.layout.button_main, linearLayout, false);
	linearLayout.addView(button);
}
```

显示效果如图：



## inflate(layoutId, null)

使用这个方法，没有指定父容器，现象就是第一个 button 并没有按照布局文件中设定的那样，显示 300dp 的高度。根据 inflate 方法中的第 79 行，直接将解析到 button 返回，我们直接拿到这个 button 是没有用的，想要在 LinearLayout 中显示出来，还需要调用 addView 的方法，但是此时的 button 是没有 LayoutParams 的，所以直接执行 addView，那么检查到该 button 没有 LayoutParams，会给其添加一个自带的 LayoutParams，在 LinearLayout 中，产生的默认 LayoutParams 如下：

```java
@Override
protected LayoutParams generateDefaultLayoutParams() {
    if (mOrientation == HORIZONTAL) {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    } else if (mOrientation == VERTICAL) {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.WRAP_CONTENT);
    }
    return null;
}
```

关于 LayoutParams，如果不了解的话，请参考[对 LayoutParams 的理解](https://zachaxy.github.io/2017/05/11/%E5%AF%B9LayoutParams%E7%9A%84%E7%90%86%E8%A7%A3/)

我们的线性布局是纵向的，所以宽度是父容器宽度，高度是包裹内容，结果正是我们看到的第一个 button 的效果；



## inflate(layoutId, root )

使用这个方法，指定了父容器为 LinearLayout，inflate 方法中如果使用了两个参数的方法，那么第三个参数 attachToRoot 的值默认为 true

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

此时 root 不为 null 所以 attachToRoot 为 true，满足 inflate 方法中第 73 行的条件，所以为 button 的 LayoutParams 与 LinearLayout 进行关联，从而满足了 button 的 300dp 的高度，结果就是现实的第二个 button 的效果。



## inflate(layoutId, root, false )

使用这个方法，root 不为空，但是 attachToRoot 为 false，所以调用了 inflate 方法中第 52 行的逻辑，为这个 view 添加了 root 的 LayoutParams，并返回该 view，之前也说过，一个 View 如果没有与父容器关联的 LayoutParams，那么这个 view 是没有办法使用的。所以在这里要比第二种方法多了一步手动 addView 的方法，所以这样和第二种效果是一样的，只是 attachToRoot 的值，如果为 true，inflate 时自动帮我们添加到 root 中，为 false，那么返回的 view 需要我们手动执行一次 root.addView 方法才可以，当然这里我们也可以将 view 解析出来之后，不调用 root 的 addView 方法，而是调用其它 ViewGroup 的 addView 方法，但是这样感觉有点混乱，如果想调用其它 ViewGroup 的 addView 方法，何不在 inflate 时，将 root 设置为我们真正的 ViewGroup。



# 在 inflate 方法中炸出的几个方法

## createViewFromTag

> 该方法的作用是根据解析到的布局文件的节点名（eg：RelativeLayout，TextView，com.your.coustom.view 等）创建对应的 Java 对象，在 Java 中如何根据类名创建对象呢？很显然是用反射来实现的。这就是 createViewFromTag 所完成的功能，其将解析到对应的 View 返回。



那么接下来深入源码，看一下其具体流程：

```java
final View temp = createViewFromTag(root, name, inflaterContext, attrs);


private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
	return createViewFromTag(parent, name, context, attrs, false);
}


View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,boolean ignoreThemeAttr) {
	if (name.equals("view")) {
		name = attrs.getAttributeValue(null, "class");
	}

// Apply a theme wrapper, if allowed and one is specified.
	if (!ignoreThemeAttr) {
		final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
		final int themeResId = ta.getResourceId(0, 0);
	if (themeResId != 0) {
		context = new ContextThemeWrapper(context, themeResId);
		}
	ta.recycle();
	}

	if (name.equals(TAG_1995)) {
   		 // Let's party like it's 1995!
   	 	return new BlinkLayout(context, attrs);
    }

    try {
    	View view;
        if (mFactory2 != null) {
       		 view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
        	view = mFactory.onCreateView(name, context, attrs);
        } else {
        	view = null;
   		 }

    if (view == null && mPrivateFactory != null) {
    	view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    if (view == null) {
   		 final Object lastContext = mConstructorArgs[0];
   		 mConstructorArgs[0] = context;
    try {
    	if (-1 == name.indexOf('.')) {
    	view = onCreateView(parent, name, attrs);
    } else {
   		 view = createView(name, null, attrs);
   		 }
    } finally {
   		 mConstructorArgs[0] = lastContext;
    	}
    }

    return view;
    } catch (InflateException e) {
		throw e;
	} catch (ClassNotFoundException e) {
		final InflateException ie = new InflateException(attrs.getPositionDescription()
			+ ": Error inflating class " + name);
          ie.initCause(e);
          throw ie;

	} catch (Exception e) {
		final InflateException ie = new InflateException(attrs.getPositionDescription()
			+ ": Error inflating class " + name);
		ie.initCause(e);
		throw ie;
	}
}
```

第 30 行，开始解析我们的 view 了，这里是使用不同的 mFactory 来创建 View，优先顺序为：mFactory2=》mFactory=》mPrivateFactory，哪个 Factory 不为空，就用哪个，调用其 createView 方法，通过反射创建相应的对象，createView 方法就不在往下跟了，要不这篇就解不了了，后面有时间会写一篇关于插件换肤的文章，会对 createView 方法进一步深入。



总之，你要记住结论，那就是 createViewFromTag 方法根据布局文件解析到的节点名，通过反射创建相应的对象。

## rInflate

> 回顾调用 rinflate 方法的地方，是在解析布局文件时，根 tag 为"merge"，我们知道 merge 必须是某个布局文件的根 tag，其下包含的所有的子 view 们直接添加到其父容器中，这样就减少了一层布局的嵌套。在单独的一个布局文件中，merge 的表现类似于 FrameLayout。那么根据这个特性，解析时直接跳过 merge 这一层，进入内部，继续解析具体的 view。



```
rInflate(parser, root, inflaterContext, attrs, false);


void rInflate(XmlPullParser parser, View parent, Context context,
		AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

	final int depth = parser.getDepth();	//这个 depth 的值，估计是 2，因为 merge 作为根标签，其 depth 是 1，现在进入了 merge 的内部，所以为 2
	int type;

	while (((type = parser.next()) != XmlPullParser.END_TAG ||
			parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

		if (type != XmlPullParser.START_TAG) {
			continue;
		}

		final String name = parser.getName();
		
		if (TAG_REQUEST_FOCUS.equals(name)) {
			parseRequestFocus(parser, parent);
		} else if (TAG_TAG.equals(name)) {
			parseViewTag(parser, parent, attrs);
		} else if (TAG_INCLUDE.equals(name)) {
			if (parser.getDepth() == 0) {
				throw new InflateException("<include /> cannot be the root element");
			}
			parseInclude(parser, context, parent, attrs);
		} else if (TAG_MERGE.equals(name)) {
			throw new InflateException("<merge /> must be the root element");
		} else {
			final View view = createViewFromTag(parent, name, context, attrs);
			final ViewGroup viewGroup = (ViewGroup) parent;
			final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
			rInflateChildren(parser, view, attrs, true);
			viewGroup.addView(view, params);
		}
	}

	if (finishInflate) {
		parent.onFinishInflate();
	}
}
```

第 7 行，拿到当前标签的深度，其深度在这里是 2，因为已经跳过了跟标签 merge，进去其内部了，xml 的深度示意图如下：

```xml
<!-- outside -->     0
<root>               1
  sometext           1
    <foobar>         2
      ...			 2
    </foobar>        2
</root>              1
<!-- outside -->     0
```

第 23 行，主要这里解释了如果解析到的 tag 是 include，那么并且其深度为 0，那么直接抛出异常，这表明 include 标签是不能作为根标签的。只能作为子标签，接下来开始调用 parseInclude 方法解析 include 指定的布局文件，内部依旧是类似于 createViewFromTag 的方法通过反射不断的创建 view 对象，这里不再深入。

第 31 行，这里调用了 createViewFromTag 的方法，创建了 merge 标签下的 view，并将该 view 添加到 rinflate 方法的第二个参数 root 中，其调用了 root.addView(view)，如果该 view 是一个 ViewGroup，其内部还有其它子 view，会调用 rInflateChildren 方法，继续解析该 view 下的子 view，最终结果相当于直接将 merge 标签下的子元素添加到 merge 标签 parent 中，这样就保证了不会引入额外的层级。



## rInflateChildren

> 该方法被调用的时机是在当解析到一个 ViewGroup 类型的标签时（eg：LinearLayout 等），那么会调用该方法，继续解析该 ViewGroup 下的标签。其内部的实现很简单，直接调用了上面的 rinflate 方法，以当前解析的到 ViewGroup 作为该方法的第二个参数 root，进行解析。

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```





# 总结：解析 XML 布局文件的作用

本节以 LayoutInflater 方法的不同参数抛出问题，引出 LayoutInflater 内部的执行流程，并解释了 inflate 方法中的几个解析布局文件的关键方法，然后又根据 inflate 不同的参数，观察其不同的现象。相信此刻你对于 LayoutInflater 的使用以及其内部的执行原理已经很清楚了。接下来再做一个总结



当我们用 LayoutInflater 解析一个布局文件的时候到底做了什么，简单的一句话就是：**将布局映射成一棵多叉树**。

inflate 方法的核心就是根据不同的标签，反射出不同的 View 对象。那么采用何种方法，才能表现 xml 布局文件中，父容器中和子 View 的包含关系呢？

答案就是使用了 ViewGroup#addView 方法，如果解析的 View 是一个 ViewGroup，那么会将该 ViewGroup 作为 parent，继续解析内部的 view，并将其添加到了 ViewGroup#mChilddren 数组中，这个数组默认长度为 12，如果子 view 数量过多，会对该数组进行扩容，都是以 12 进行扩充。如果子 View 依然是一个 ViewGroup，那么继续执行上面的过程，这个过程就是**深度优先遍历**，将整个布局文件映射为一棵树，每个父节点必定是一个 ViewGroup，其对应的子节点就是在该 ViewGroup 标签下的 View，其引用保存在该 ViewGroup 的 mChildren 数组中，这样就形成了一棵多叉树，其根节点就是布局文件的根节点。



最后附上一张inflate的内部执行流程图

![inflate方法内部执行流程图](http://opda1q948.bkt.clouddn.com/LayoutInflater.png)