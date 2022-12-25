---
title: 手写JVM系列(5)-分析class文件-常量池
date: 2017-05-09 10:30:01
tags: JVM
---

> 常量池占据了 class 文件很大一部分数据，里面存放着各式各样的常量信息，包括数字和字符串常量、类和接口名、字段和方法名等。本节将详细介绍常量池和各种常量。

# 常量池在 class 文件中格式

```java
ClassFile {
  ...
  u2 constant_pool_count;	//常量池大小
  cp_info constant_pool[constant_pool_count-1]; //常量池
  ...
}
```



首先读出常量池的大小，这是一个 u2 类型的数据，因此，这也表明了常量池最大为 2^16-1，如果你不是故意为难虚拟机，将工程中所有的代码都写到一个 class 文件中，相信这已经足够用了。



接下来根据这个大小,生成常量信息数组，注意：

1. 表头给出的常量池大小比实际大 1,所以这样的话,虽然可能生成了这么大的,但是 0 不使用,直接从 1 开始;
2. 有效的常量池索引是 1~n–1。0 是无效索引，表示不指向任何常量
3. CONSTANT_Long_info 和 CONSTANT_Double_info 各占两个位置。也就是说，如果常量池中存在这两种常量，实际的常量数量比 n–1 还要少，这也导致了 1~n–1 的某些数也会变成无效索引。




# 常量池分类

常量池中的常量分为两类：字面量 和 符号引用

- 字面量:文本字符串,声明为 final 的常量值等
  - 数字常量

  - 字符串常量

    ​
- 符号引用:Javac 在编译的时候,没有像 c 语言那样的"连接"这一步,而是在加载 class 文件的时候动态加载,拿到该符号引用指向的字符串,再使用反射,加载相应的类.
  - 类
  - 接口
  - 字段
  - 方法

字面量是可以直接获取到其值的,而符号引用是通过索引直接或者间接指向 CONSTANT_Utf8_info 常量,然后拿到其字面量的;



# 常量池具体类型

常量池具体可以分为以下 14 中类型，其基本结构都是由一个 u1 的 tag，后面跟一个具体类型的数据组成。

![常量池中的 14 种常量项的结构总表](http://oi8e3sh89.bkt.clouddn.com/image/jvm/%E5%B8%B8%E9%87%8F%E6%B1%A0%E4%B8%AD%E5%B8%B8%E9%87%8F%E9%A1%B9%E7%BB%93%E6%9E%84%E8%A1%A8.png)

按照常量池的分类一节，将这 14 中分为两类

- 字面量：
  - CONSTANT_Integer
  - CONSTANT_Float
  - CONSTANT_Long
  - CONSTANT_Double
  - CONSTANT_Utf8
- 符号引用：剩余的 10 种。

下面分别从字面量大类和符号引用大类中各选两个有代表性的常量进行讲解。



##  CONSTANT_Integer

CONSTANT_Integer_info 使用 4 字节存储整数常量，其结构定义如下：

```java
CONSTANT_Integer_info {
  u1 tag;
  u4 bytes;
}
```

因此我们在读到 tag 为 CONSTANT_Integer 时，接下来需要读取 4 字节的数据，从而拼接成这一个整数。

float，long，double 和 int 是类似的，读到对应的 tag，分别去读 4,8,8 字节的数据拼接起来，就是实际的数值。



## CONSTANT_Utf8

CONSTANT_Utf8 类型代表的是常量池中真正的字符串，其结构定义如下，读取到对应的 tag 后，后面是一个 u2 的数据，表明字符串的长度 len，接下来是长度为 len 的字节，代表真正的字符串。

```java
CONSTANT_Utf8_info {
  u1 tag;
  u2 length;
  u1 bytes[length];
}
```

但是这里要注意的是，字符串在 class 文件中是以 MUTF-8（Modified UTF-8）方式编码的。MUTF-8 编码方式和 UTF-8 大致相同，但并不兼容。
差别有两点：一是 null 字符（代码点 U+0000）会被编码成 2 字节：0xC0、0x80；二是补充字符（Supplementary Characters，代码点大于U+FFFF 的 Unicode 字符）是按 UTF-16 拆分为代理对（Surrogate Pair）分别编码的。具体转换方法请看[源码](https://github.com/zachaxy/JVM/blob/master/Java/src/classfile/ConstantUtf8Info.java)，这里的转换方法是直接根据 java.io.DataInputStream.readUTF（）方法改写的。将读到的 byte 数组经过转码之后得到其代表的具体字符串。



常量类型中有一个 ConstantStringInfo，这个常量的名字迷惑性比较强，但是它里面并没有保存真正的字符串，而是保存了一个指向常量池中的索引，这个索引对应的常量类型一定是 CONSTANT_Utf8，这个常量中的字符串才是 ConstantStringInfo 中想表达的字符串。



## CONSTANT_Class

CONSTANT_Class_info 常量表示类或者接口的符号引用，指向是接口或者类名。

```java
CONSTANT_Class_info {
  u1 tag;
  u2 name_index;
}
```

其 name_index 是常量池索引，指向 `CONSTANT_Utf8_info` 常量。所以如果想真正的拿到当前 class 的全限定名，需要通过 name_index 先得到常量池中的 `CONSTANT_Utf8_info `，然后在获取其中的值。



## CONSTANT_NameAndType

CONSTANT_NameAndType_info 给出字段或方法的名称和描述符。CONSTANT_Class_info 和 CONSTANT_NameAndType_info 加在一起可以唯一确定一个字段或者方法。其结构如下：

```java
CONSTANT_NameAndType_info {
  u1 tag;
  u2 name_index;
  u2 descriptor_index;
}
```

字段或方法名由 name_index 给出，字段或方法的描述符由 descriptor_index 给出。name_index 和 descriptor_index 都是常量池索引，指向 CONSTANT_Utf8_info 常量。字段和方法名就是代码中出现的（或者编译器生成的）字段或方法的名字。



Java 虚拟机规范定义了一种简单的语法来描述字段和方法，可以根据下面的规则生成描述符。

1. 类型描述符。

   - 基本类型 byte、short、char、int、long、float 和 double 的描述符是单个字母，分别对应 B、S、C、I、J、F 和 D。注意，long 的描述符是 J 而不是 L。
   - 引用类型的描述符是“L＋类的完全限定名＋分号”
   - 数组类型的描述符是“[＋数组元素类型描述符”
2. 字段描述符就是字段的类型描述符。
3. 方法描述符格式是：“（按参数顺序的参数类型描述符）+返回值类型描述符”，其中 void 返回值由单个字母 V 表示。




# 常量池类型总结

可以把常量池中的常量分为两类：字面量（literal）和符号引用（symbolic reference）。字面量包括数字常量和字符串常量，符号引用包括类和接口名、字段和方法信息等。除了字面量，其他常量都是通过索引指向 CONSTANT_Utf8_info 常量来获取其对应的字符串。


