---
title: MOP——方法注入
date: 2017-12-10 11:45:09
tags: Groovy
---

前面[MOP——方法拦截](https://zachaxy.github.io/2017/12/09/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E6%8B%A6%E6%88%AA/)介绍了利用 MOP 对方法的调用进行拦截，接下来要介绍利用 MOP 实现方法的注入。

# 方法拦截和方法注入的区别
拦截：侧重对于**已有的方法**的调用进行拦截
注入：对一个已有的类**添加新方法**，以拓展该类的功能。

eg：Java 中提供了 String 的类，如果我们想扩展该类，为其提供一个字符串加密方法。在 Java 中，最常见的做法是提供一个接口，内有 encrypt() 方法，让目标类实现该接口；或者继承该类，然后添加一个 encrypt() 方法。但是这里存在的问题是：我们未必可以修改想要扩展的类，就像 String，还是 final 的，只能用，不能改。

不过使用 Groovy，就可以方便的为任何类扩展方法，同时在使用起来，给人的感觉就好像注入的类是该类本身就有的。

MOP 的注入有四种实现方式：
1. 分类(Category)
2. ExpandoMetaClass
3. Minxin
4. trait

<!--more-->

# 使用分类进行方法注入
第一次接触 Category 的概念是在学习 Objective-C 的时候，只要自定义一个和目标类相同的类，然后在自定义类中添加方法，那么在方法调用时，会先从自定义的类中查找，找不到后再去原本的类中查找。这样一来不仅可以扩展类的方法，同时还可以覆盖原有类的方法。Objective-C 中 Category 感觉是最优雅的方式了。而 Groovy 中的 Category 就逊色的多，接下来看一下 Groovy 中 Category 的使用方法，这里以向 String 中添加一个 encrypt 方法为例。
```groovy
class StringUtils {
    def static encrypt(String self) {
        byte[] arr = self.bytes;
        for (int i = 0; i < arr.length; i++) {
            arr[i] = (127 - arr[i])
        }
        return new String(arr)
    }


}

class IntegerUtils{
    def static add(Integer a, int b) {
        a + b
    }
}

use(StringUtils,IntegerUtils) {
    String str = "hello"
    s = str.encrypt()
    println 1.add(2)
}
```

这里先定义了一个的类，其中定义了一个方法 encrypt(),要想要该类成为 String 的分类，需要注意以下几点：
1. 其内部定义的方法必须为 `static`
2. 方法的第一个参数必须定位为目标类的类型(eg：这里定义的 String，当然你可以不写类型，这样就有可能让多个类都是用该方法了),如果该方法还需要参数，那么就从第二个形参开始声明。
3. 第一个参数如果声明类型，必须为包装类的类型，eg：如果我们想为整数提供方法，即使用到了 `1.add(2)` 这样的调用方式，但这是 groovy 提供的语法糖，其本质任为 Integer，因此在定义分类的方法时，第一个参数必须是包装类型


在使用时，其必须在 use 所定义的代码块中，出了代码块就无法使用分类中的方法了，否则报找不到方法的错误。在 use 后面必须注明要注入的方法所在的类，eg：use(StringUtils)，use 中可以注入多个类，如果多个分类中有重复的方法定义，那么以最后一个分类中方法为准。

# 使用 ExpandoMetaClass 进行方法注入

## 注入概述
之前其实我们已经见到过使用 ExpandoMetaClass 注入方法的示例了，就是使用[MetaClass 进行方法拦截]()，这本质就是方法的注入，只不过注入的方法名(invokeMethod)比较特殊，成为了方法拦截。同样，我们也可以用 ExpandoMetaClass 对类进行其它方法的注入，还拿上面 Integer 的加法的例子：
```groovy
Integer.metaClass.add = {
    int i ->
        delegate + i
}

println 1.add(3)
```

## 注入的种类
使用 ExpandoMetaClass 方法注入，可以对以下三种方法进行注入：
- 非静态方法
- 静态方法
- 构造器
- 属性

接下来一一介绍如何注入：
1. 非静态方法注入
  这在前面已经见到过了，也是最常用的注入，使用方法：
  ```groovy
  Foo.metaClass.bar = {}
  foo.bar()
  ```
  Groovy 的设计理念就是让程序的编写更加流程，因此在 DSL 中，可能更常见的一种形式是在调用方法时不写括号，即`foo.bar`但是没有括号调用时，会将方法的调用当成属性，所以需要对之前的注入进行修改。
  ```groovy
  Foo.metaClass.getBar = {}
  foo.bar
  ```
  这样的调用方式是否更加优雅呢，在后面的 DSL 中，还会进一步讲解 groovy 的语法糖，让编程更加优雅。
2. 静态方法注入
  需要使用 `'static'` 的特殊字面量注入静态方法
  ```groovy
  Foo.metaClass.'static'.bar = {}
  Foo.bar()
  ```
3. 注入构造器
  使用 `constructor` 属性注入构造器
  添加一个构造器 <<
  替换一个构造器 =
  ```groovy
  Foo.metaClass.constructor << { 
  	int i -> 
  	Foo foo = new Foo();
  	foo.i = i
  	foo
  }
  ```
  构造方法注入特别要注意的是，要确保没有递归调用自身，否则栈溢出。因为我们是想定义构造器，肯定会借助现有的构造器，然后进行属性的改造，但是不要产生递归。如果是想覆盖构造器的话，那么只能在内部使用反射
4. 注入属性
  类似以闭包的方式注入方法，属性注入也是支持的，只要在后面 = 具体值 即可。
  ```groovy
  Foo.metaClass.bar = 1
  println foo.bar
  ```
5. 一次注入多个方法
  Groovy 提供了使用`ClassName.metaClass.method = { ... }`这样的语法向 metaClass 中添加，既简单又方便，但如果想添加一堆方法，这样的声明就会感觉很费劲。groovy 提供了更简洁的语法，用来减少噪音！！这种方式也是在 DSL 中常见到的。
  ```groovy
  Foo.metaClass = {
  	bar1 = {}
  	bar2 = {}

  	'static'{
  		bar3 = {}
  	}
  	//针对于不管是覆盖还是注入，在这种语法环境下，都应该使用 = 
  	constructor = {
  		int i - >
  	}

  	constructor = {
  		int i,int j ->

  	}
  }
  ```


**再次重申：使用 ExpandoMetaClass 注入的闭包中，delegate 指的是调用该方法的对象，在此基础上，闭包中使用类原本的成员变量，或者方法也是可以的。**

## 向单个实例中注入方法
前面介绍的是向整个类中注入方法，那么基于该类的所有对象都可以使用闭包中的方法。如果只是想扩展该类的某一个对象的方法，而不影响该类的其它对象，该如何处理呢？
其实不单是 Class，每个具体的对象也包含一个 metaClass，我们可将创建一个具体的 ExpandoMetaClass 实例，并将制定方法加入其中，然后将其赋给对应的具体对象，也可以将方法直接注入到具体的对象的 metaClass 上。

```groovy
//方式 1
class Man{
	def talk(){}
}
def emc = new ExpandoMetaClass(Man)
emc.sing = { -> ... }
emc.initialize()
def mike = new Man()
mile.metaClass = emc
mike.sing()

//方式 2
mike.metaClass.dance = { -> ...}
mike.dance()


//卸载之前注入的方法
mike.metaClass = null
```
很明显方式 2 是最为优雅的，因此推荐使用方式 2。同时，当我们为一个对象注入了方法，在使用了一段时间不想使用后，那么很方便的卸载之前注入的方法。


## ExpandoMetaClass 小结
使用 ExpandoMetaClass，无论是注入方法，还是调用方法，都比 Category 要优雅的多。因此推荐使用该方法。
但是要注意的是，如果对象想使用注入的方法，必须要先进行注入。如果在已经有对象产生之后再向类中注入方法，那么该对象无法调用注入的方法！！
因此使用 ExpandoMetaClass 进行注入，最好是在整个应用初始化时进行。

同时方法注入具有继承性。如果向 Object 注入了方法，那么所有的类都可以使用该方法。




# 使用 Minxin，trait 进行方法注入
这两种方式更像是开头提到的定义接口的实现方式。
个人感觉最为强大的方式是 Mixin 的方式。可以为类注入多个 Mixin，就好想让类实现了多个接口，同时接口中相同的方法，以后面加入的为准。
这里不再重点展开了。
[Groovy Mixin 注入](http://blog.csdn.net/chennai1101/article/details/56282305)
[Groovy 2.3 introduces traits](https://www.mscharhag.com/groovy/groovy-introduces-traits)
[Mixins and traits](https://www.ibm.com/developerworks/library/j-jn8/index.html)

 

# 实现方式的优劣对比
- Category 存在的问题:其作用被限定在 use()块内，所以也就限定于当前执行的线程。进入该 use()块内的代码会在当前线程创建一个栈帧，并压入到当前线程的栈上，而当 use 代码块结束后，当前线程的栈会将刚刚压入的栈帧弹栈。但是如果频繁的调用 use 代码块，势必会对性能造成一定的影响。

  凡事都有两面性，Categoty 的使用 use 块，提供了更好的隔离性，我们可以再不同的地方，使用不同的分类，这也为类的扩展提供了灵活性。

- trait：缺点是在有类的修改权的情况下才能使用，类似接口。

- Mixin：其实是最强大的方式，但需要对其有进一步的了解，以防走火。。

这里推荐使用 **ExpandoMetaClass。**