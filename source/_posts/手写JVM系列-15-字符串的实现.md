---
title: 手写JVM系列(15)-字符串的实现
date: 2018-01-16 20:52:01
tags: JVM
---


Java 程序中最常用的除了基本类型的变量，就数得上字符串了吧，String 在 Java 中并不是基本类型，而是一个引用类型，但是它又很特别，所以特意对 JVM 中的 String 进行讲解。但是不得不承认的是，我的示例中是用 java 来构建的 JVM，起初的目的就是为了快速实现该 JVM，快速的了解 JVM 内部原理，虽然达到了目标，但是在实现该 JVM 内部的一些细节时，总有些不尽人意的地方，这是当初所没想到的，例如本节字符串池的实现，以及后面本地方法的实现上，所写的代码都显得有点挫。不过对于理解 JVM 的内部原理，依然有很大帮助。

<!--more-->

# 字符串在不同位置所使用的编码
回想整个 JVM 对字符串处理的过程，有两个个关键的地方对字符串进行了处理。下面的讲解中出现了几种编码，如果你对编码不是很了解，请先阅读[理解各种字符编码](https://zachaxy.github.io/2018/01/15/%E7%90%86%E8%A7%A3%E5%90%84%E7%A7%8D%E5%AD%97%E7%AC%A6%E7%BC%96%E7%A0%81/) ，以便后续内容的理解。

## class 文件中的字符串常量->JVM
class 文件中解析常量池时，在 class 文件中字符串常量是以 `MUTF-8`编码的形式保存的，我们读取文件时，需要进行额外的处理。

所谓的额外处理就是说当我们把常量池中`ConstantUtf8Info`类型的常量加载到内存中，该字符串在 JVM 中用哪种编码来表示。这其实取决于我们的 JVM 是用什么语言来写的，如果我们用 go 语言来实现 JVM，go 语言中的字符串是用`utf-8`的编码，所以 class 文件中的字符串常量被加载到 JVM 中之后，其表现形式就是 `utf-8`；而我的 JVM 是用 java 语言来写的，java 语言中字符串是用`unicode`的编码，所以 class 文件中的字符串常量被加载到 JVM 中之后，其表现形式就是 `unicode`；无论是`utf-8`还是`unicode`，都需要将`MUTF-8`格式的字节按照目标格式进行转换，才可以被正确的使用，否则就是乱码。如果你用的是另一种语言来写 JVM，而该语言中字符串的编码恰好为`MUTF-8`，那么你就不用进行转换，直接读取字符串到内存就好了。

可不幸的是，没有哪种编程语言中字符串是用的`MUTF-8`编码。这种编码格式也太不常用了，目前也搞不清楚为何 class 文件中要单独用 `MUTF-8`编码来保存字符串。既然是用 java 实现的 JVM，那么在解析 class 文件中的字符串常量时，我们是进行了编码的转换的：具体代码在[ConstantUtf8Info.java 中](https://github.com/zachaxy/JVM/blob/master/Java/src/classfile/classconstant/ConstantUtf8Info.java)
```java
public class ConstantUtf8Info extends ConstantInfo {
	@Override
	void readInfo(ClassReader reader) {
	    int len = reader.readUint16();
	    byte[] data = reader.readBytes(len);
	    try {
	        val = decodeMUTF8(data);
	    } catch (IOException e) {
	        e.printStackTrace();
	    }
	}

	//将 MUTF8 转为 UTF8 编码, 根据 java.io.DataInputStream.readUTF（）方法改写。
	private static String decodeMUTF8(byte[] bytearr) throws IOException {
	    ......
	    return new String(chararr, 0, chararr_count);
	}
}
```

## JVM->Java 应用程序
上面一步是将 class 文件中常量池的字符串加载到 JVM 中了，但这还不是最终目的，我们最终的目的是在应用程序中使用该字符，一个最简单的例子：
```java
void f(){
	String s = "hello";
	System.out.println(s);
}
```
上述方法中，我们的目标就是打印出 s 所代表的“hello”字符串。要能正确的打印出“hello”，首先要明白，我们在代码中写了“hello”这个字符串之后，通过`javac`的处理，“hello”会存储到 class 文件中了。我们真正用的是 class 文件。那么上一步已经把 class 中的“hello”加载到 JVM 中了，接下来程序要使用真实的 java 程序中，其 String 是用`unicode`来表示的，因此还需要将 JVM 中的“hello”转换为`unicode`编码。万幸的是，我们的 JVM 是用 java 来实现的，所以从 JVM 中获取对应的字符串，本身就已经是`unicode`编码了，无须任何转换就可以实现，但是这也是我所不满的地方，因为真实的 JVM 并不是用 java 来实现的，所以这里无法具体模拟具体转换的细节了（后续可能会考虑用 c++ 对本虚拟机进行重构，那么这个问题自然就遇到了。）

而底层真正实现这一转换是在下面即将介绍的 ldc 指令中实现的！




# class 文件常量池中的 ConstantStringInfo 常量 VS ConstantUtf8Info 常量
在[分析 class 文件中的常量池](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-5-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B8%B8%E9%87%8F%E6%B1%A0/)中，我们介绍了`ConstantStringInfo`和`ConstantUtf8Info`这两个常量，但是其区别我们并没有做出说明，放着这一节进行说明，可能更合适。
我们知道 ConstantUtf8Info 常量是真正存放字符串的，而 ConstantStringInfo 常量中并没有字符串，而是保存了一个常量池的索引，该索引指向的正是一个 ConstantUtf8Info 常量。不知道你有没有想过为什么会有一个 ConstantStringInfo 的常量这么麻烦，要多一步才能获取到真正的字符串，全部都用 ConstantUtf8Info，省去中间一步的过程，岂不更快？

这里就涉及到 class 文件中`ConstantUtf8Info`常量保存的字符串是什么内容了，可以分为两类：
- 类名，方法名，变量名，描述符，这些字符串是用来确定加载哪个类，调用哪个方法，使用哪个变量的，是供 JVM 内部使用的。
- 在 java 程序中显示声明的 String 类型的变量，是供 java 程序使用的。

我们写 java 程序其实只关心我们定义的 String 类型的变量，而 String 类型的变量在 class 文件中就是用 `ConstantStringInfo`常量来表示的。这就解释了为什么都是表示字符串，使用两种常量进行区分。这里主要是把 JVM 使用的字符串和 java 应用程序使用的字符串所区分开。

两类字符串的赋值时期也是不同的，JVM 内部使用的像类名，方法名，描述符等字符串是在 class 文件转为 Zclass 对象时，进行的赋值。而`ConstantStringInfo`常量是在 class 文件中的常量池转换为 运行时常量池时进行的转换，此时 RuntimeConstantPool 中的字符串就不在是再通过索引指向另一个常量了，而是保存的真正的常量。下面是常量池转换时，对`ConstantStringInfo`常量的处理：
```java
case ConstantInfo.CONSTANT_String:
    //在对字符串引用进行转换的时候，转为字符串直接引用
    ConstantStringInfo stringInfo = (ConstantStringInfo) classFileConstantInfo;
    this.infos[i] = new RuntimeConstantInfo<String>(stringInfo.getString(), ConstantInfo.CONSTANT_String);
    break;


//这里用到了 ConstantStringInfo 的 getString()方法，其源码为：
public String getString() {
    return constantPool.getUtf8(stringIndex);
}
```


# String 变量详解
前面介绍`ConstantStringInfo`常量时，我说过在 java 代码中定义的 String 类型的变量，都会在 class 文件中保存为`ConstantStringInfo`常量，但是那个表述并不准确。并不是代码中任何一个 String 类型的变量，都会被保存在 class 文件中的`ConstantStringInfo`常量中。
我们看如下两个 String 类型的变量，已经其对应的字节码

1. 直接定义字符串
```java
void f0(){
	String s0 = "a#4$";	
}

//该方法对应的字节码
0 ldc #3 <a#4$>
2 astore_1
3 return
```

通过查看字节码，我们发现如果是定义的直接字面量"a#4$"的 String 类型的变量，那么该字符串"a#4$"就会在 class 文件中保存为`ConstantStringInfo`常量，同时在运行时，被转为运行时常量池中的常量，ldc 指令表示将当前运行时常量池中索引为 3 的变量压入操作数栈，随后再将该变量保存到本地变量表索引为 1 的位置。

结论：String 类型的变量，在赋值时使用的是直接字面量，则会在 class 文件中保存为`ConstantStringInfo`常量。

1. 通过字符串连接符`+`定义字符串
```java
void f1(){
	String s1 = "a#" + "4$";
}

//该方法对应的字节码
0 ldc #3 <a#4$>
2 astore_1
3 return
```
通过查看字节码，我们发现用字符串连接符`+`定义字符串，最终同样是想定义"a#4$"的字符串，其对应的字节码和直接定义"a#4$"是完全一样的，之所以会产生这样的结果，完全是编译器帮我们做的优化。

结论：使用字符串连接符`+`定义字符串，如果连接的子串也是直接字面量，那么编译器会帮我们做优化，其结果和定义连接后的字面量的字符串结果是完全一样的。


1. 通过变量定义字符串
```java
void f2(){
	String tmp = "4$";
	String s2 = "a#" + tmp;
}

//该方法对应的字节码
 0 ldc #3 <4$>
 2 astore_1
 3 new #4 <java/lang/StringBuilder>
 6 dup
 7 invokespecial #5 <java/lang/StringBuilder.<init>>
10 ldc #6 <a#>
12 invokevirtual #7 <java/lang/StringBuilder.append>
15 aload_1
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 invokevirtual #8 <java/lang/StringBuilder.toString>
22 astore_2
23 return
```

通过查看字节码，我们同样是定义一个最终字符串为"a#4$"的变量，但是使用字符串连接符时，一个子串使用变量替代了之前的直接字面量，发现字节码有很大的不同，同样是编译器在背后做了手脚，其发现如果一个通过`+`定义的字符串中，如果子串有变量，那么编译器会偷偷的产生一个 StringBuilder 类，而`+`符号也会被转换为 StringBuilder 的 append 方法。最终拼接好的字符串是刚刚的 StringBuilder 调用了其 toString 方法，而查看 StringBuilder 的 toString 方法，其内部是重新创建了一个 String 的对象：
```java
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}
```

结论：使用字符串连接符`+`定义字符串，如果连接的子串中包含变量，那么该字符串不会在 class 文件中保存为`ConstantStringInfo`常量，而是通过 StringBuilder 在方法运行时动态生成的一个全新的 String 变量。

# 字符串池 StringPool
通过上述三种定义字符串的创建过程，我们可以发现一个现象，虽然最终字面量是一样的，但是如果用`==`比较结果可能是不一样的。
```java
String s1 = "a#4$";

String s2 = "a#" + "4$";

String tmp = "4$";
String s3 = "a#" + tmp;

System.out.println(s1 == s2);  //true
System.out.println(s1 == s3);  //false
System.out.println(s1 == s3.intern());  //true
```
通过该程序的结果，我们可以发现 s1 和 s2 都是直接从运行时常量池相同的位置获取的值，所以其用 `==`比较得到的结果是 true，因为本来就是同一个对象。但 s3 并不是从运行时常量池直接取的值，而是在方法内部通过 StringBuilder 的 toString 方法获取的一个新的 String 对象，所以和 s1 比较，自然得到的结果就是 false 了。

这里要注意的是：在运行时常量池保存的字符串类型的常量，其实保存的并不是字符串本身，而是一个 key ！这个 key 是 JVM 中 StringPool 的一个 key，由该 key 对应的 value 得到的值才是真正 Java 中 String 变量。

这样设计的原因是：String 是一个类，该类中保存了字符串的值，本质上是一个 char 类型的数组，但是 String 和 char[] 又不是对等的，char[] 是 String 的主要组成部分，但是 String 毕竟是一个类，其内部还有其它的成员和方法来操作自身的 char[]。同时 String 又是一个不可变类，也就是说如果一个 String 类型的变量确定之后，其内部的 char[] 的元素是不可以改变的，是不可变的(只读的)，同时 String 类型的变量在程序中又很常用，所以在 JVM 中定义了一块内存，叫做字符串池——StringPool。注意这个 StringPool 是整个应用程序所共享的，里面存放的就是在 class 文件中被定义为`ConstantStringInfo`常量所对应的真正的字符串，而在运行时常量池中保存的所谓的字符串常量，其实只是 StringPool 的一个引用。这样设计的目的，可以达到一个应用中所有相同的字符串常量是唯一的，从而达到节约 JVM 内存空间的目的。

观察字符串赋值的语句可以发现，在上例的(1)和(2)中，都是使用 ldc 指令，将运行时常量池中获取到字符串常量，注意：此时从运行时常量池获取到的字符串为 JVM 中的字符串，而不是 java 应用程序中的编码，真正实现这一转换过程的是在 ldc 命令中进行的转换，具体实现过程可参考 [LDC.java](https://github.com/zachaxy/JVM/blob/master/Java/src/instructions/constants/LDC.java) 

这里简述一下 StringPool 的实现过程，其内部是一个 HashMap，key 为 JVM 中的字符串，value 为 java 引用程序中可是使用的 String，注意这个 String 是指包含了 char[] 的一个 Object，而不单单是 char[]。如果 key 相同，那么得到的 Object 就是同一个。当然，如果当前 map 中不存在 key，那么就添加该 (key,value)对。
这也就解释了为什么上例中 s1==s2。同时 s3 对应的 Object 并不不是从 StringPool 中获取的，而是在当前 Frame 中重新 new 的一个对象。所以 s1!=s3。


同时注意到 `s1==s3.intern()`。String 中有一个 intern 方法，该方法的作用是以 s3 的字面量为 key，从 StringPool 中寻找对应的值，如果不存在则添加，并返回对应的 object。因为 s3 的字面量和 s1 是相同的，所以从 StringPool 中返回值的话，得到的 object 和 s1 是一样的。

具体实现请参考 [StringPool.java](https://github.com/zachaxy/JVM/blob/master/Java/src/runtimedata/heap/StringPool.java) 
但是这里的实现，并不如意。原因还是本 JVM 是采用 java 实现的，导致字符串在 JVM 中的编码和 java 应用程序中的编码是一样的，这一个转换有些多此一举，但是为了模拟 StringPool 的功能，还是添加了对应的代码。同时要注意，String 在 JVM 中也是一个 Zobject，通过 key 获取到的不应该是一个 java 中的 String，而应该是 JVM 中的 Zobject。
