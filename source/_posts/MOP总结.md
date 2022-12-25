---
title: MOP总结
date: 2017-12-12 08:57:42
tags: Groovy
---

之前介绍了基于 MOP的技术：
- [MOP——方法拦截](https://zachaxy.github.io/2017/12/09/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E6%8B%A6%E6%88%AA/)
- [MOP——方法注入](https://zachaxy.github.io/2017/12/10/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E6%B3%A8%E5%85%A5/)
- [MOP——方法合成](https://zachaxy.github.io/2017/12/11/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E5%90%88%E6%88%90/)

接下来对 MOP 的这三种技术的使用做一个汇总

<!--more-->


# 在运行时创建类。
经过之前的讲解，我们可以发现，正是 MOP 的出现，使得 Groovy 变得非常灵活，可以改变已有类的结构，拦截或添加方法等，这为我们的开发工作提供了极大的便利性。既然可以扩展已有的类，那么我们也可以借助 MOP 来动态的创建一个类。这里需要借助 MOP 为我们提供的而一个类——Expando 类。

```groovy
man = new Expando()
man.name = 'zx'
man.age = 18
man.work = { "working" }

println man
println man.work()
```

Expando 像是一个可以容纳任何属性的类，我们可以为其添加任意的属性，任意的方法(以闭包的形式)，其实无论是属性，还是闭包，都被保存在 Expand 中的 Map 中，key 对应属性\闭包名，value 对应属性值，或者闭包体


# 关于拦截/注入/合成
经过之前三篇的学习，我们了解到，无论是拦截，注入，还是合成。每种技术都有若干实现方式。但是都不可避免的用到 MetaClass 这个类。拦截时，向其中注入了 invokeMethod 方法；合成时，向其中注入了 methodMissing 方法。注入的都是闭包，但是调用时如果找不到对应的方法，那么就会从闭包中找对应的名字，同时闭包的参数要和对应的方法相同。

- 对于拦截，如果我们能修改类，就实现 GroovyInterceptable 接口，并实现 invokeMethod 方法；否则就在 ClassName.metaClass 上注入 invokeMethod
- 对于合成，如果我们能修改类，就在类中重写 methodMissing 方法；否则就在 ClassName.metaClass 上注入 methodMissing
- 对于注入，如果注入的方法在之前的类中已经实现过了，同时又在 ClassName.metaClass 上进行了注入，那么 metaClass 上的方法会优先执行。

相信到这里在看这幅图应该更容易理解：
![Groovy interception mechanism](http://docs.groovy-lang.org/next/html/documentation/assets/img/xGroovyInterceptions.png.pagespeed.ic.lQZRl0FSbC.webp)

# 一个综合应用
之前将的 MOP 的三大技术是分开来讲的，现在把这三个技术融合起来，实现这样一个场景，对于一个类，我们调用其方法，如果存在，就执行其本身的方法，如果不存在就进行方法合成。

```groovy
class Man {
	def work(){
		"I can work"
	}
}

Man.metaClass.invokeMethod = {
	String name,args ->
		def method = Man.metaClass.getMetaClass(name,args)
		if(method){
			method.invoke(delegate,args)
		}else{
			Man.metaClass.invokeMissingMethod(delegate,name,args)
			//或者执行下面的方法，metaClass 中如果没有 invokeMethod 方法，那么会调用 metaClass 中的 methodMissing 方法，但是不如上面直接调用来的直观
			//Man.metaClass.invokeMethod(delegate,name,args)
		}
}

Man.metaClass.methodMissing = {
	String name,args ->
    def skills = ["run","sing","dance","talk"];

    if (skills.contains(name)){
    	def impl = {
    		Object[] vars ->
         "I can " + name
    	}

    	Man.metaClass."$name" = impl

    	impl(args)
    }else{
    	throw new MissingMethodException(name,Man.class,args)
    }
```

在这个例子中，首先使用拦截技术，将所有的方法调用拦截住(MetaClass 注入 invokeMethod 方法)，然后分析本类是否可以执行该方法，如果可以路由给原本的方法，如果不可以那么执行合成(MetaClass 注入 methodMissing 方法)。

这里注意到一个细节：我们在方法合成之后，先将合成的方法注入到 MetaClass 中，这样在第二次调用的时候，就不会在此合成了，而是直接调用，大大提高的执行效率
