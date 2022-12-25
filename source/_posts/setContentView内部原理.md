---
title: setContentView内部原理
date: 2017-05-13 23:49:53
tags: Android Framework
---

> 我们平时在写Android界面的时候，一般都是创建一个Activity（eg：MainActivity.java），并创建一个对应的布局文件（eg：activity_main.xml），然后在Activity的onCreate方法中，调用setContentView(R.layout.activity_main)，这样布局文件中所展示的界面就能显示出来了。相信刚开始学习Android时，这一步骤是让我们的界面显示出来的必经套路，那么其内部究竟执行了什么，才能实现这一效果？今天就一起来了解一下setContentView方法的内部实现原理。

# 早期的setContentView方法

之前我们在编写Android程序的时候，创建的XXXActivity默认都是继承Activity这个类的，

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().requestFeature(Window.FEATURE_NO_TITLE);
        setContentView(R.layout.activity_main);
    }
}
```

进入setContentView方法中一探究竟：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

可以看到其实Activity中的setContentView方法自己并没有执行上面逻辑，而是调用了与Activity关联的Window的setContentView，



# 现在的setContentView方法

```java
public class MainActivity extends AppcompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().requestFeature(Window.FEATURE_NO_TITLE);
        setContentView(R.layout.activity_main);
    }
}
```
