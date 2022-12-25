---
title: 手写JVM系列(4)-分析class文件
date: 2017-05-09 09:49:14
tags: JVM
---

> 前面一节已经介绍了如何获取了字节文件的字节流,那么在获取到字节流之后,就要读取并解析字节码了,这一节会介绍 class 文件的结构.本节的代码均在**classfile包**下,源码[在这里](https://github.com/zachaxy/JVM)



#  class文件的基本结构

构成 class 文件的基本数据单位是字节，可以把整个 class 文件当成一个字节流来处理。稍大一些的数据由连续多个字节构成，这些数据在class文件中以**大端（big-endian）方式存储**。为了描述class文件格式，Java虚拟机规范定义了`u1`、`u2`和`u4`三种数据类型来表示1、2和4字节**无符号整数**

class文件中相同类型的多条数据一般按表（table）的形式存储（包括接下来要讲的常量池，属性表，接口索引集合，字段表集合，方法表集合）,表由表头和表项（item）构成，表头是 u2 或 u4 整数。假设表头是 n，后面就紧跟着 n 个表项数据。

class文件的结构描述:

```java
ClassFile {
  u4 magic;	//魔数
  u2 minor_version;	//次版本号
  u2 major_version;	//主版本号
  u2 constant_pool_count;	//常量池大小
  cp_info constant_pool[constant_pool_count-1]; //常量池
  u2 access_flags;	//类访问标志,表明class文件定义的是类还是接口，访问级别是public还是private，等
  u2 this_class;	//
  u2 super_class;	//
  u2 interfaces_count;	//本类实现的接口数量
  u2 interfaces[interfaces_count];	//实现的接口,存放在数组中
  u2 fields_count;		//本来中含有字段数
  field_info fields[fields_count];	//数组中存放这各个字段
  u2 methods_count;		//本类中含有的方法数
  method_info methods[methods_count];	//数组中存放着各个方法
  u2 attributes_count;			//本类中含有的属性数量;
  attribute_info attributes[attributes_count];	//数组中存放着各个属性
}
```



# 解读 class 文件

接下来依次对 class 文件中的各个字段做一个介绍:

## 魔数
> 很多文件格式都会规定满足该格式的文件必须以某几个固定字节开头，这几个字节主要起标识作用，叫作魔数（magic number）。例如 PDF 文件以 4 字节“%PDF”（0x25、0x50、0x44、0x46）开头，ZIP 文件以 2 字节“PK”（0x50、0x4B）开头。class 文件的魔数是“0xCAFEBABE”。
> 开头的四字节,起标识作用.Java 虚拟机规范规定，如果加载的 class 文件不符合要求的格式，Java 虚拟机实现就抛出`java.lang.ClassFormatError`异常。

校验魔数使用的方法:`ClassFile#readAndCheckMagic()`

## 版本号
- 次版本号(m):2 字节
- 主版本号(M):2 字节

完整的版本号可以表示成“M.m”的形式。次版本号只在 J2SE 1.2 之前用过，从 1.2 开始基本上就没什么用了（都是 0）。主版本号在 J2SE 1.2 之前是 45，从 1.2 开始，每次有大的 Java 版本发布，都会加 1。 

校验版本使用的方法:`ClassFile#readAndCheckVersion()`



## 常量池

版本号之后是常量池，但是由于常量池比较复杂，所以放到[手写JVM系列(5)-分析class文件-常量池](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-5-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B8%B8%E9%87%8F%E6%B1%A0/)中介绍。



## 访问标志

在常量池之后,紧接着是两字节的访问标志（access_flags），这个标示用来识别**类或者接口**的访问信息，两字节供 16 位，目前只定义了 8 位，没有用到的一律用 0 来填充，以备以后的拓展使用。

| 标志名称           | 标志值    | 含义                                       |
| :------------- | ------ | ---------------------------------------- |
| ACC_PUBLIC     | 0x0001 | 是否为 public 类型                            |
| ACC_FINAL      | 0x0010 | 是否被声明为 final，只有类可设置                      |
| ACC_SUPER      | 0x0020 | 是否允许使用 invokespecial 字节码指令的新语义，JDK1.0.2 之后编译出来的这个标志都必须为 1 |
| ACC_INTERFACE  | 0x0200 | 标示这是一个接口                                 |
| ACC_ABSTRACT   | 0x0400 | 是否为 abstract 类型，对于接口或者抽象类来说，此标志为真，其他类值为假 |
| ACC_SYNTHETIC  | 0x1000 | 标志这个类并非由用户代码产生的                          |
| ACC_ANNOTATION | 0x2000 | 标志这是一个注解                                 |
| ACC_ENUM       | 0x4000 | 标志这是一个枚举                                 |

 



## 类索引

访问标志之后是 u2 类型的类索引数据，用于确定这个类的全限定名。该 u2 类型的索引值指向常量池中一个类型为 CONSTANT_Class_info 的类描述符常量,再通过 CONSTANT_Class_info 类型的常量中的索引值,可以找到定义在 CONSTANT_Utf8_info 类型的常量中的全限定名字符串。

类索引说白了就是一个指向常量池的索引值，那么直接用一个 int 值来表示即可。



## 父类索引

类索引之后是 u2 类型的父类索引数据，用于确定这个类的父类的全限定名。由于 Java 语言不允许多继承，所以其父类索引只有一个。除了 java.lang.Obect 之外,所有的 Java 类都有父类，所以除了 java.lang.Obect 的父类索引为 0，其余都不为 0。

该 u2 类型的索引值指向常量池中一个类型为 CONSTANT_Class_info 的类描述符常量,在通过 CONSTANT_Class_info 类型的常量中的索引值,可以找到定义在 CONSTANT_Utf8_info 类型的常量中的全限定名字符串。



同理，父类索引也是直接用一个 int 值来表示。





## 接口索引集合

父类索引之后是 u2 类型数据的接口索引集合。用来表述这个类实现了哪些接口。按照 implements 语句后面的接口顺序从前向后在接口索引集合中。

入口的第一项是 u2 类型的接口计数器，表示接口索引表的大小，如果该类没有实现接口，则计数器为 0，后面的接口索引表不占用任何字节。



因为有可能实现了多个接口，所以使用一个数组来盛放实习了接口在常量池中的索引值。



## 字段表集合

接口索引集合之后是字段表集合。虚拟机规范给出的字段结构定义如下，字段表用来描述类或接口中声明的变量。包括静态变量和非静态变量。：

```java
field_info {
  u2 access_flags;		//字段的访问修饰符
  u2 name_index;		//常量池索引，代表字段的简单名称
  u2 descriptor_index;	//常量池索引，代表字段描述符
  u2 attributes_count;	//字段的额外附加属性数量
  attribute_info attributes[attributes_count];	//字段的额外的附加属性
}
```



字段表中第一个字段 u2 是访问修饰符，这和前面讲的类访问符标志很相似，下表列举下字段的访问标志可选项：

| 标志名称          | 标志值    | 含义              |
| ------------- | ------ | --------------- |
| ACC_PUBLIC    | 0x0001 | 字段是否为 public    |
| ACC_PRIVATE   | 0x0002 | 字段是否为 private   |
| ACC_PROTECTED | 0x0004 | 字段是否为 protected |
| ACC_STATIC    | 0x0008 | 字段是否为 static    |
| ACC_FINAL     | 0x0010 | 字段是否为 final     |
| ACC_VOLATILE  | 0x0040 | 字段是否为 volatile  |
| ACC_TRANSIENT | 0x0080 | 字段是否为 transient |
| ACC_SYNTHETIC | 0x1000 | 字段是否由编译器自动产生    |
| ACC_ENUM      | 0x4000 | 字段是否是 enum 类型   |



在实际情况中访问标志的使用时有限制的，比如 ACC_PUBLIC、ACC_PRIVATE 和 ACC_PROTECTED 三个标志只能选其一，ACC_FINAL 和 ACC_VOLATILE 只能选其一等，这些都是由 Java 本身的语言规范所决定的。

随着 access_flags 标志后的两项索引值是 name_index 和 descriptor_index。他们都是对常量池的引用，分别代表着字段的名称和描述。

字段的简单名称：不含类型的字段名称。 （eg：int  i,其简单名称为 i）

字段的描述符：用来描述字段的数据类型。(eg: int i,其描述符为 I)

​									描述符标识字符含义:

![](http://i.imgur.com/G01YtQJ.png)



对于数组类型，每一维度将使用一个前置的“[”字符来描述。如定义一个`String[][] `类型的二维数组，其描述符为：`[[Ljava/lang/String;`

用描述符来描述方法时，按照先参数列表，后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“()”之内。



在 descriptor_index 之后跟随着一个 u2 类型的数据 len，描述后面一个长度为 len 的属性表数组，这个数组用于存储一些额外的信息，字段都可以在属性表中描述零至多项的额外信息。当然这个属性数组中保存的并不是真正的属性，而是属性表的索引。关于属性表，将在[手写JVM系列(6)-分析class文件-属性表](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/)中单独介绍。



**注：字段表集合中不会列出从超类或者父接口中继承而来的字段，但是有可能列出原来 Java 代码中不存在的字段，譬如在内部类中为了保持外部类的访问性，会自动添加一个指向外部类实例的字段。**



## 方法表集合

字段表之后是方法表集合，和字段表集合的描述几乎是一样的（因此我在代码中对方法表和字段表使用同一个类 MemberInfo 来表示的），其结构如下：

```java
field_info {
  u2 access_flags;		//方法的访问修饰符
  u2 name_index;		//常量池索引，代表方法的简单名称
  u2 descriptor_index;	//常量池索引，代表方法描述符
  u2 attributes_count;	//方法的额外附加属性数量
  attribute_info attributes[attributes_count];	//方法的额外的附加属性
}
```





方法表的第一个字段是访问修饰符，和字段表访问修饰符类似，不同的是：因为 volatile 和 transient 关键字不能修饰方法，所以方法表的访问标志中没有这这两项，但是添加了 synchronized，native，strictfp，abstract 等可用来修饰方法的关键字，具体如下：

| 标志名称             | 标志值    | 含义                 |
| ---------------- | ------ | ------------------ |
| ACC_PUBLIC       | 0x0001 | 方法是否为 public       |
| ACC_PRIVATE      | 0x0002 | 方法是否为 private      |
| ACC_PROTECTED    | 0x0004 | 方法是否为 protected    |
| ACC_STATIC       | 0x0008 | 方法是否为 static       |
| ACC_FINAL        | 0x0010 | 方法是否为 final        |
| ACC_SYNCHRONIZED | 0x0020 | 方法是否为 synchronized |
| ACC_BRIDGE       | 0x0040 | 方法是否是由编译器产生的桥接方法   |
| ACC_VARARGS      | 0x0080 | 方法是否接受不定参数         |
| ACC_NAVIVE       | 0x0100 | 方法是否为 native       |
| ACC_ABSTRACT     | 0x0400 | 方法是否为 abstract     |
| ACC_STRICTFP     | 0x0800 | 方法是否为 strictfp     |
| ACC_SYNTHETIC    | 0x1000 | 方法是否由编译器自动产生       |



随着 access_flags 标志后的两项索引值是 name_index 和 descriptor_index。他们都是对常量池的引用，分别代表着方法的名称和描述。

方法的简单名称：不含类型和返回值的方法名称。

方法的描述符：描述方法的参数列表（数量，类型，顺序）和返回值。

以  eg：int func (int i，String s) 为例子,其简单名称为 func，方法描述符为：可参照上面的描述符标识字符含义图。

方法的简单名称：func

方法的描述符：(ILjava/lang/String)I



在 descriptor_index 之后跟随着一个 u2 类型的数据 len，描述后面一个长度为 len 的属性表数组，这个数组用于存储一些额外的信息，方法可以在属性表中描述零至多项的额外信息。和字段不同的是，字段只需要一个变量名和对应值就可以了，但是方法内部是包含代码的，这段代码在字节码中是如何表示的呢？答案是放在了方法属性表中的 Code 属性中，和字段表一样，方法表中的属性数组中并不保存真正的属性，而是保存的属性表的索引，关于属性表，将在[手写JVM系列(6)-分析class文件-属性表](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/)中单独介绍。



**注意：如果父类方法在子类中没有被重写，方法表集合中就不会出现来自父类的信息。和字段表类似的是，方法表中可能会出现编译器会自动添加的方法，最典型的就是：类构造器`<clinit>`和实例构造器`<init>`**



关于方法的重载

 重载一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名。特征签名就是一个方法中各个参数在常量池中的字段符号引用集合，而返回值不会包含在特征签名中。但是如果两个方法具有相同的名称和特征签名，只是返回值不同，那么也可以合法共存于同一个 Class 文件中的。但是编译器是会阻止这一行为的，如果两个方法仅仅是返回值不同，编译器会直接报错。所以只能通过字节注入的方式，实现返回值不同的两个方法。



## 属性表

方法表集合之后是属性表，但是由于属性表比较复杂，所以放到[手写JVM系列(6)-分析class文件-属性表](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/)中介绍。