---
title: MOP——方法拦截
date: 2017-12-09 17:14:35
tags: Groovy
---

前面[Groovy 对象和 MOP](https://zachaxy.github.io/2017/12/08/Groovy%E5%AF%B9%E8%B1%A1%E5%92%8CMOP/)简单了介绍了 Groovy 所提供的 MOP 机制。
接下来要介绍利用 MOP 实现方法的拦截。
拦截：在对方法进行调用时，我们可能有一些其它的要求，eg：日志的记录，执行时间的记录等，在 Java 可以 AOP 对方法的调用进行拦截，插入我们想执行的方法。而 Groovy 提供的 MOP 机制可以更为轻松的实现对方法的拦截。


MOP 的拦截有两种实现方式：
1. 实现 GroovyInterceptable 接口进行拦截
2. 利用 MetaClass 进行拦截

<!--more-->

## 实现 GroovyInterceptable 接口进行拦截
对于一个想要拦截方法的 Groovy 对象，要先实现 GroovyInterceptable 接口，然后务必重写其中的 invokeMethod()方法。我们拦截处理的逻辑都在这个方法中实现。
接下来在调用该类的**任何方法**的时候，都会被路由到 invokeMethod 方法中。我们可以在 invokeMethod 方法中，根据被拦截的方法名，来加入我们要记录的内容，然后将方法在路由到具体实际的方法中。

```groovy
class Man implements GroovyInterceptable {
    def talk() {
        System.out.println "hello"
    }

    def run() {
        System.out.println "running"
    }

    def invokeMethod(String name, args) {
        System.out.println("start call method: " + name);
        def targetMethod = Man.metaClass.getMetaMethod(name, args)
        targetMethod.invoke(this,args)
        System.out.println("end call method: " + name);

    }
}

man = new Man()

man.talk()
man.run()
```

结果：
```
start call method: talk
hello
end call method: talk
start call method: run
running
end call method: run
```

man 调用的所有方法都被拦截到了 invokeMethod 方法中。

这里要注意的有两点：
1. 不仅我们调用的方法会被拦截，方法内使用的 Groovy 上的方法(eg：`println()`方法)同样也会被拦截。所以这里使用了`System.out.println()`方法，因为其是 Java 上的方法，不会受到拦截的影响，我们这个例子很简单，如果拦截的方法内部使用了 Groovy 中的方法，那么很可能会产生递归调用，导致栈溢出。
2. 在 invokeMethod 方法中，利用类似反射的手段，根据方法名找到对应的方法，其返回值在 Groovy 中是 MetaMethod，其类型和 Java 中的 Method 类型相似，这里用 targetMethod 表示返回值，但是这里必须显示的使用 def 定义 targetMethod,否则报错`groovy.lang.MissingPropertyException: No such property: targetMethod for class: Man` 原因可参考[深入理解 Android（一）：Gradle 详解](http://www.infoq.com/cn/articles/android-in-depth-gradle)中`脚本中的变量和作用域`一节的讲解。

额(⊙o⊙)…坑好多

## 利用 MetaClass 进行拦截
使用 MetaClass 重写上述的方法
```groovy
class Man {
    def talk() {
        System.out.println "hello"
    }

    def run() {
        System.out.println "running"
    }
}


Man.metaClass.invokeMethod = {
    String name, args ->
        System.out.println("start call method: " + name);
        targetMethod = Man.metaClass.getMetaMethod(name, args)
        targetMethod.invoke(delegate, args)
        System.out.println("end call method: " + name);
}

man = new Man()

man.talk()
man.run()
```
其结果和实现 GroovyInterceptable 接口的结果相同

这里要注意的是：
1. 这里将一个闭包赋给了 Man.metaClass 的 invokeMethod，也就是间接的实现了 invokeMethod 方法。因为 groovy 中调用方法时，如果找不到对应的方法名，会从该类的闭包中查找是否有相同的名字，若该闭包的参数也符合，那么就会直接调用该闭包。如果你一时接受不了这种方式，你可以重新创建一个 MetaClass 的实例，重写 invokeMethod 方法，然后将自己实现 metaClass 赋给 Man.metaClass，两种方式是等价的，但是明显前者的方式更简洁。
2. 实现的闭包中一定注意其有两个参数。否则依然无法实现拦截
3. 在闭包中，不用为 targetMethod 加 def 的定义
4. 执行方法时，调用 invoke 方法，第一个参数要传 delegate。因为这是在闭包中。


**所有形如 Foo.metaClass.bar = { delegate } 的闭包中出现的 delegate，该 delegate 代表的就是调用 bar 方法的 Foo 实例对象**


## 两种拦截方法的使用场景
第一种需要实习接口，这就需要我们可以修改该类的源码才能让该类实现接口，否则只能使用第二种方法。