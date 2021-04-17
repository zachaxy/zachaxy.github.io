---
title: groovy入门
date: 2017-10-26 22:16:20
tags: Groovy
---

# 变量定义

groovy 是动态类型的语言，也就是说不需要指定变量的类型，类型是可以值来推导的。

- 不指定类型定义变量： `i = 10`
- 使用关键字 `def`，虽不是必须的，但是为了代码清晰，还是建议使用 def 关键字定义变量 `def i = 10`

这里有个误区：def 关键字的出现时替代变量类型的占位符，如果你已经明确了变量的类型，就没必要使用 def，也就是说 `def int i = 10` 是没有必要的

# 字符串

- 单引号:单引号中的内容严格对应 Java 中的 String
- 双引号:和脚本语言类似，如果字符串中有$符的话，先对 $表达式进行求值
- 三引号:针对的是很长的字符串，只要在三引号之间，可以随意换行

```groovy
def s1 = 'hello'

def s2 = "string append ${version + 1}"

def s3 = '''随意
换行
打印
'''
```

字符串的长度：`s1.size()`

字符串的访问：`s1[2]`   //被当做数组来处理；

或者  `s1.getAt(index)`

字符串的重复：

`s = s1*3`  // 代表是将 s1 拼接三次

# 函数定义

- 参数可以不指定类型
- 方法体中可以不使用 return 来返回，这样默认的返回值就是最后一行语句的值，但是如果不指定 return，定义函数时必须要使用 def。
- 可以指明返回值类型,这样就可以省略 def 关键字；但是如果未指定返回值类型，那么定义函数时必须使用 def 关键字
- 默认参数：可以给参数定义默认值，但是要注意的是，如果要定义有默认值的参数，那么该参数必须在参数列表的末尾

```groovy
int sum(int a,int b = 0){
	return a + b;
}

sum(1,2)
sum(1)
```

# 函数调用

- 调用时，可以不加括号，但这并不是一个好的习惯，因为如果一个函数是无参的，那么不带括号，很可能被误认为是属性的调用；同时在嵌套函数调用时，也容易出错（将参数设置为另一个方法调用的结果）
- 最好的一种形式是：能加括号都加括号；如果该方法只有一个参数，那么调用时加括号，如果有多个参数，则可以省略括号
- 特别注意：如果该函数没有参数，那么调用时，必须加括号；同时这个的参数个数要和闭包的参数个数区分开

# Groovy bean 对象

```groovy
class  Bean{
    int version
    String name;

    Bean(int version,String name){
        this.version = version;
        this.name = name;
    }
}

Bean bean1 = new Bean(1,"zx")

println bean1.name
```

上面定义的 Bean 对象，在其内部定义了两个成员变量 version 和 name，这样默认就会为我们生成其对应的 getter 和 setter 方法。而在外部访问该成员变量时，其实是调用对应的 getter 和 setter 方法。



# 闭包

闭包和函数是类似的，是一个代码片段，可以定义一个引用指向这个闭包，这类似于 C 语言中的函数指针；

闭包还可以获取外部变量；与 Java 中的内部类不同的是，在闭包内不仅可以获取到外部变量，同时也可以将其修改，修改对外部是可见的。

与函数定义不同的是，闭包的参数是在方法体内定义的，以 -> 分割，左侧为参数，右侧为方法体

闭包方法体内，最后一句为返回值。

闭包的默认参数：如果定义的闭包只有一个参数，那么可以在闭包内直接写方法体，使用 `$it` 来获取参数。

如果要定义一个没有参数的闭包，那么闭包内， -> 左侧为空，则代表该闭包没有参数

Groovy 中的函数，可以用闭包作为参数；

> 之前在看 groovy 的代码,或者写 groovy 的代码,调用外部定义的闭包,总是不清楚这个闭包到底要传入几个参数. 其实可以根据 IDE 的提示,如果是一个纯的大括号,那么必然是属于只有一个参数的闭包



# 集合类

groovy 中的集合包含两种：list 和 map，其使用也很简单

```groovy
list = [0,1,2]
list << 3
println list.size();

map = [name:"zx",age:25];
map.merry = false	// 向 map 添加键值对
println map.size()  // 3
```



# 运算符

- 范围运算符(..)

```groovy
//range 更像是一个数组，其长度为 6，其为从 5 到 10
def range = 5..10
//通过 get(index)来获取对应的值
println range.get(2)

// 其它实例
1 .. <10    //  独占范围的示例
'a'..'x'   // 范围也可以由字符组成
10..1     // 范围也可以按降序排列
'x'..'a' // 范围也可以由字符组成并按降序排列。

// 由此就有如下 foreach 的写法
for(int i in 1..5) {
	println(i);
}
```

- 三目运算符(?:)

```
// 假设有这样一个需求，如果 name 不为 null，那么就输出 name 的值，否则输出 Anonymous
// 下面的三目运算符的写法，未免有些啰嗦
displayName = name ? name : 'Anonymous'

// 下面是 groovy 中支持三目运算符的写法
displayName = name ?: 'Anonymous'
```

- 展开操作符(*.)

应用于实现了 iterator 的对象中，一般是数组，集合，是 foreach 的缩写

```groovy
def technologies = ['Groovy','Grails','Gradle']
technologies*.toUpperCase() // = to technologies.collect { it?.toUpperCase() }
```

- 安全操作符(?.)

可以有效的避免空指针

```groovy
// eg：get(1)返回的结果是 null，此时如果调用 user.username 会产生空指针异常
//而是用 ?. 如果 user 为空,那么 username 也为 空，不会产生空指针异常
def user = User.get(1)
def username = user?.username
```

# 文件操作

groovy 提供了比 Java 更为便捷的方法来操作文件，其[API 文档参考](http://www.groovy-lang.org/gdk.html)

```groovy
File file = new File("E:/hello.txt")
//按行读文件 
file.eachLine {  
   line -> println "line : $line"; 
} 

//读取文件的整个内容，是用 text 属性
println  file.text

//获取文件内同的大小，单位是字节
println file.length()

// 写入文件,内容为 hello world
file.withWriter('utf-8') { 
	writer -> writer.writeLine 'Hello World' 
}  

//复制文件
def src = new File("E:/hello.txt")
def dst = new File("E:/hello1.txt")
dst << src.text

//递归显示目录及其子目录中的所有文件
file.eachFileRecurse() {
	file -> println file.getAbsolutePath()
}
```



# 参考

[官方手册](http://groovy-lang.org/documentation.html)
[Learn groovy in Y minutes](https://learnxinyminutes.com/docs/groovy/)
[使用 Groovy 开发之新特性](http://www.jianshu.com/p/ba55dc163dfd)
[日积月累--Groovy 语言规范之操作符](http://blog.csdn.net/dabaoonline/article/details/50477090)
[Groovy 学习之-运行时元编程](http://www.jianshu.com/p/2c17a50ff7f1)
[精通 Groovy](https://www.ibm.com/developerworks/cn/education/java/j-groovy/j-groovy.html#N1036D)