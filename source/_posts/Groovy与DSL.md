---
title: Groovy与DSL
date: 2017-12-13 10:33:46
tags: Groovy
---

DSL：领域特定语言。常用于聚焦指定的领域或问题，这就要求 DSL 具备强大的表现力，同时在使用起来要简单。由于其使用简单的特性，DSL 通常不会像 Java，C++等语言将其应用于一般性的编程任务。

由语言的难易程度，做一个简单的排序：配置文件 < DSL < 编程语言(eg:Java,C++...)
- 配置文件是最简单的一种，常见的有用 xml 来进行配置项目，这需要预先定义好一些规范，只有按照预先定义好的规范书写配置文件，才能得到正确的解析。解析其实也是使用编程语言，通过读文件的形式，按照预先定义的规范，得到相应的配置。
- 编程语言是最高级的，其本身可以实现条件判断，循环等这样的逻辑，这是配置文件很难实现的。但是很少有用编程语言来直接表述一项任务的，因为其缺乏像配置文件那样简介的语义。
- DSL 则介于两者之间，即有像配置文件那样清晰的语义，即很强的语义表现力，同时自身也具备一定的逻辑。

然而 DSL 的应用领域非常有限，没有必要单独为其开发一款语言。就像 xml 中的配置文件是通过高级语言解析而来，我们也可以用高级语言解析 DSL 文件，执行相关的逻辑。但是与 xml 配置文件不同的是，xml 配置文件需要预先定义规范。而 DSL 不需要定义规范，其语法受到现有语言的约束，不用使用任何解析器，而是巧妙的讲语法映射到底层语言中的方法和属性，这样使得用户在使用起来可能意识不到自己正在使用的是一门更广泛的语言的语法。


DSL 的两个显著特点：
- 上下文相关
- 流畅的调用

鉴于 Groovy 动态特性和元编程能力，其很适合作为 DSL 底层的解析语言。接下来探讨在 Groovy 的基础上构建 DSL。

对于 Groovy 来说，一个伟大的 DSL 产物就是新一代构建工具——Gradle。


<!--more-->

# 利用闭包实现上下文相关性
上下文：就是一个特定的语境，在这个语境下，大家没有必要一直重申这一语境前提，而是所有的逻辑都在这个前提下执行。
看如下 Groovy 代码：
```groovy
list = [1,2]
list.add(3)
list.add(4)
println list.size()
```
在上面的代码中，每条语句都是针对 list 的，但是每条语句都重复声明了 list 的语境，未免有些啰嗦，于是 groovy 提供了 with()方法，用来设置上下文，上面的代码可以修改为：

```groovy
list = [1,2]
list.with{
	add(3)
	add(4)
	println size()
}
```
这里使用了闭包，将闭包中的语句的上下文路由到 list。原因是 with 中的闭包将其 delegate 设置到了调用 with 的对象上。

同时注意到这里 with 的原型是 `with(closure)`，当一个方法的最后一个参数是闭包，同时我们传入的是一个匿名闭包时，那么其调用的方式就是上述 with 的形式，这不是方法的定义，而是方法的调用，只不过后面的闭包是作为方法的最后一个参数，单独拿出来了。单独拿出来比写在括号内，语法更加简洁。因为花括号后面在跟一个右括号总感觉破坏了代码的优美性。

受到 Groovy 所提供的 with 的启发，我们也可以仿照 with 的实现方式，提供类似的上下文执行环境，实现我们的 DSL。
```groovy
def addList(closure){
	list = []
	closure.delegate = list
	closure()
	list
}

// 调用 addList 提供的上下文特性，想 list 中添加元素
list = addList(){
    add(1)
    add(2)
    add(3)
    size()
}

println list
```

通过将闭包的代理进行配置，自然就提供了一个统一的上下文环境，这样我们在闭包中调用的任何方法，都是基于之前设置的代理的，这一方法在 gradle 中是很常见的。


# Groovy 针对流畅性的先天优势
流畅性可以提高代码的可读性，使代码自然流动。

Groovy 所具备的天然的优势有：
- 动态类型和可选类型
- 动态加载，操作和执行脚本的灵活性
- 分类和 ExpandoMetaClass 可以扩展已有类的功能
- 闭包为执行提供很好的上下文
- 操作符的重载可以使我们自由的定义操作符


## 灵活的方法调用语法
如果方法是**有返回值**的，那么在调用时可以不使用点符号(.) 而是用空格将调用者和方法分开，就能在返回的这个实例上进行连续的调用

## 灵活的括号语法
当调用的方法需要参数时，Groovy 不要求使用括号，若有多个参数，那么参数之间依然使用逗号分隔；如果不需要参数，那么方法的调用必须显示的使用括号。

## 让无参数的方法调用也不需要括号
为了和有参数的方法调用时不使用括号的语法，统一风格，我们设法让无参数的方法调用也不需要括号。当然这个功能是 Groovy 自身不支持的。直接这样调用，Groovy 会认为这是在访问一个属性，那么我们结合 Groovy 中对属性的访问就是对 getXXX 的访问，将无参数的方法名改成 getXXX 的形式，即可实现“调用无参数的方法不需要括号”的语法！

结合刚刚提到的三点，我们设计一个 DSL 计算器：
```groovy
value = 0
def getClear() { value = 0}
def add(number) { value += number }
def getTotal() { println "Total is #value" }

//接下来开始使用这一简介的 DSL
clear
add 2
add 5
total
```

由上面的 groovy 创建的简易加法器的 DSL，可以感受到其强大的流畅性。我们在调用时，甚至都可能意识不到这是代码，而是一些配置文件或者普通的文字描述，但这确实是 Groovy 代码。

接下来提供一组这样的数据，进一步解读 DSL。
```groovy
jump fast,forward turn right move ahead
```

上面的依旧是 Groovy 代码，首先是 jump 方法的调用，其接受两个参数 fast 和 forward，该方法的返回值类型上有一个 turn 方法，其接受一个 right 参数，同时方法的返回值类型上有一个 move 方法，其接受一个 ahead 参数。这就是 DSL 强大的表现力。其底层就可以用之前讲到的 Groovy 特性提供技术支持。

## MOP 与 DSL
接下来使用 MOP 中的方法合成

```groovy
detailInfo = [:]
def methodMissing(String name,args){
    detailInfo[name] = args
}

def introduce(closure){
    closure.delegate = this
    closure()
    println "hello,everyone"
    detailInfo.each{
        key,value ->
            println "My $key is $value"
    }
}


introduce{
    name "zx"
    age 18
}
```

可以看到，利用 Groovy 的 MOP 能力，设计与执行 DSL 将变得非常容易。

