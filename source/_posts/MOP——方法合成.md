---
title: MOP——方法合成
date: 2017-12-11 09:27:16
tags: Groovy
---

前面[MOP——方法注入](https://zachaxy.github.io/2017/12/10/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E6%B3%A8%E5%85%A5/)介绍了利用 MOP 对方法的调用进行注入，接下来要介绍利用 MOP 实现方法的合成。

合成：在运行期，根据输入状态的不同，动态的生成一个方法。
这里要注意的是：方法合成，在运行时动态的生成方法，看起来很智能，但是合成是有前提的，就是调用的方法的方法名必须要符合预先定义好的规范，才能生成。


同样，这里要和前面的方法注入区分开：
方法的合成和注入看起来似乎相同，都是向类中添加方法，以扩展类的功能，但这里也是有区别的：
- 注入是已经知道要注入的方法名了，是发生在编译期的。
- 合成是在编译期还不知道方法名，只有在调用的时候，发现并没有对应的方法，所以动态的生成该方法，然后并调用。
  合成强调的是在运行时，注入强调的是在编译期；

实现方法合成有以下两种方式：
- 使用 methodMissing 合成方法
- 使用 ExpandoMetaClass 合成方法



<!--more-->



# 使用 methodMissing 合成方法

首先明确以下方法合成的使用场景：其发生在对象调用了一个自身并不存在的方法，正常情况下会抛出异常，但是我们想根据调用的方法动态去生成一个对应的方法并执行。那么生成对应的方法这个步骤需要首先汇总到一个方法中——methodMissing，然后在该方法中，实现我们合成方法的逻辑。还是要注意，不是随意调用一个方法都能合成的，没有那么只能，必须符合预先设定好的规则。例如我们预先为一个人设定好其技能列表，但是不生成具体的方法，因为不知道外界会不会调用这些方法，如果全部实现一遍，到时候又可能用不到，所以需要在运行时，动态生成。

```groovy
class Man {
    def skills = ["run","sing","dance","talk"];

    def methodMissing(String name,args){
        if (skills.contains(name)){
            println "I can " + name
        }
    }
}

man = new Man()
man.dance()
```

# 使用 ExpandoMetaClass 合成方法
当我们无法编辑类的源文件，那么就无法使用上面的方法了，对于这类情况，依然可以使用 ExpandoMetaClass 来合成方法，在[MOP——方法注入](https://zachaxy.github.io/2017/12/10/MOP%E2%80%94%E2%80%94%E6%96%B9%E6%B3%95%E6%B3%A8%E5%85%A5/)中有提到过向 ExpandoMetaClass 中来注入方法，这里我们依然可以将 methodMissing 方法注入到 ExpandoMetaClass，从而实现方法合成。

```groovy
class Man {
}

Man.metaClass.methodMissing = { String name,args ->
    def skills = ["run","sing","dance","talk"];

    if (skills.contains(name)){
        println "I can " + name
    }
}


man = new Man()
man.dance()
```

# 为具体实例合成方法
就像为具体实例注入方法一样，我们也可以为具体实例合成方法，其代码也是类似的

```groovy
class Man{
}

def emc = new ExpandoMetaClass(Man)
emc.methodMissing = { String name,args ->
    def skills = ["run","sing","dance","talk"];

    if (skills.contains(name)){
        println "I can " + name
    }
}
emc.initialize()
def mike = new Man()
mike.metaClass = emc
mike.sing()
```