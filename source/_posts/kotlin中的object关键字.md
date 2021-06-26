---
title: kotlin中的object关键字
date: 2021-06-12 18:49:18
tags: [编程语言,kotlin]
---

> object 关键字: 用来定义一个类同时创建一个实例(声明即定义)
- 定义单例
- 实现类似 java 中静态变量
- 实现类似 java 中静态方法
- 对象表达式:匿名内部类的一种方式
<!--more-->




# 定义单例
由于 object 关键字的特殊性,其天然具备单例的特性
```kotlin
object Single {
    fun foo() {
        println("foo() in Single")
    }
}
```

调用方式:
``` kotlin
    Single.foo();           // 在 kotlin 中这样调用
    Single.INSTANCE.foo();  // 在 java 中这样调用
```

该单例类同样可以继承父类,或者实现接口,但是不允定义单例类有构造方法,默认为无参构造方法,否则无法实现单例


# 定义静态成员变量
kotlin 中没有 static 关键字,在 java 类中,定义静态成员变量可以在类中使用 static 关键字,而在 kotlin 中,可以在类中定义 object 对象,其用法同 java 中的 static 变量类似,只不过 kotlin 中定义即创建;
 
定义在类中的对象声明,同样可以继承父类或实现接口,并添加扩展方法

# 定义静态方法
要实现 kotlin 中静态方法的效果,需要使用两个关键字 `companion` + `object `; 其本质依然是创建了一个单例类,官方定义为伴生对象


```kotlin
class Person(val name: String) { 
    companion object Loader {
        fun foo() {
            println("person companion")
        }
    }
}

Person.foo()        // (1)
Person.Loader.foo()  // (2)

Person.Loader.foo(); // (3)
//Person.Companion.foo(); //(4)
```

1. 伴生对象类的名字可以自定义,如上面的 Loader, 也可以忽略
2. 之所以说像静态方法,是因为在 kotlin 中调用的时候,可以像(1)中的方式调用
3. 在 kotlin 中如果伴生对象有定义名字,也可以像(2)中的方式去调用,也可以忽略类名,像(1)中的方式调用; 因此 kotlin 类中最多只能有一个伴生对象!!
4. 在 Java 中访问伴生对象,如果有定义类名,eg:Loader, 则方式方式如(3)所示,如果未定义,那么其访问方式为(4),使用 Companion 获取伴生对象
5. 伴生对象同样可以可以继承父类或实现接口,并添加扩展方法


# 定义对象表达式创建匿名内部类
因为 object 定义即创建对象,这种特性也符合匿名内部类的写法:

```kotlin
/**
 * @author zhangxin
 * @date   2021/6/12
 */

interface OnClickListener {
    fun onClick()
}

open class AnimationAdapter {
    open fun onStart() {}
    open fun onEnd() {}
}


fun main() {
    object : OnClickListener {
        override fun onClick() {
            println("onClick")
        }
    }

    object : AnimationAdapter() {
        override fun onStart() {
            println("onStart")
        }
    }
}
```

特别注意的是: 使用 object 定义的匿名内部类不是单例,而是每次执行的时候都会重新创建