---
title: 手写JVM系列(6)-分析class文件-属性表
date: 2017-05-09 16:24:05
tags: JVM
---

> 和常量池类似，各种属性表达的信息也各不相同，因此无法用统一的结构来定义。不同之处在于，常量是由 Java 虚拟机规范严格定义的，共有 14 种。但属性是可以扩展的，不同的虚拟机实现可以定义自己的属性类型。由于这个原因，Java 虚拟机规范没有使用 tag，而是使用**属性名**来区别不同的属性。属性数据放在属性名之后的 u1 表中，这样 Java 虚拟机实现就可以跳过自己无法识别的属性。

# 属性的基本结构

属性的结构定义如下：

```java
attribute_info {
  u2 attribute_name_index;
  u4 attribute_length;
  u1 info[attribute_length];
}
```

第一个字段是属性名在常量池的索引，指向常量池中的 CONSTANT_Utf8_info 常量。

第二个字段是后面跟着的属性字节码的长度，用 u4 表示，表明属性的大小最大可以为 2^32-1。

第三个字段就是属性字节码。



# 属性的种类

按照用途，有 23 种预定义属性，可以分为三组。

1. 第一组属性是实现 Java 虚拟机所必需的，共有 5 种；
2. 第二组属性是 Java 类库所必需的，共有 12 种；
3. 第三组属性主要提供给工具使用，共有 6 种。（这组属性是可选的，也就是说可以不出现在 class 文件中。如果 class 文件中存在第三组属性，Java 虚拟机实现或者 Java 类库也是可以利用它们的，比如使用 LineNumberTable 属性在异常堆栈中显示行号。）



这里只介绍几个常用的属性：

| 属性名                | 位置                                 | 含义                     | 分组   |
| ------------------ | ---------------------------------- | ---------------------- | ---- |
| Deprecated         | ClassFile, field_info, method_info | 被声明为 deprecated 的方法和字段 | 3    |
| Synthetic          | ClassFile, field_info, method_info | 标示方法或字段是由编译器自动生成的      | 2    |
| SourceFile         | ClassFile                          | 记录源文件的名称               | 3    |
| ConstantValue      | field_info                         | final 关键字定义的常量值        | 1    |
| Code               | method_info                        | Java 代码编译成的字节码指令       | 1    |
| Exceptions         | method_info                        | 方法抛出的异常                | 1    |
| LineNumberTable    | Code                               | Java 源代码的行号与字节码指令的对应关系 | 3    |
| LocalVariableTable | Code                               | 方法的局部变量描述              | 3    |



# 属性的具体介绍

由于属性种类较多，这里只选几个有代表性的属性进行讲解。

##  Deprecated

最简单的属性，仅起标记作用，不包含任何数据。Deprecated 属性用于指出类、接口、字段或方法已经不建议使用，编译器等工具可以根据 Deprecated 属性输出警告信息。J2SE 5.0 之前可以使用 Javadoc 提供的@deprecated 标签指示编译器给类、接口、字段或方法添加 Deprecated 属性。其结构定义如下：

```java
Deprecated_attribute {
  u2 attribute_name_index;
  u4 attribute_length;
}
```

由于不包含任何数据，所以 attribute_length 的值必须是 0。自然也就没有了后面的 info 数组字段了。



##  Synthetic

最简单的属性，仅起标记作用，不包含任何数据。Synthetic 属性用来标记源文件中不存在、由编译器生成的类成员，引入 Synthetic 属性主要是为了支持嵌套类和嵌套接口。其结构定义如下：

```
Synthetic_attribute {
  u2 attribute_name_index;
  u4 attribute_length;
}
```

由于不包含任何数据，所以 attribute_length 的值必须是 0。自然也就没有了后面的 info 数组字段了。

### 桥接方法

这里要说一下，哪些方法是我们在代码里没有写，但是字节码文件会给我们加上呢? 首先构造方法，如果我们默认不写，那么字节码会自动添加上的。还有就是泛型，这里详细说明一下。

```java
abstract class A<T> {
    abstract T get(T t);
}

class B extends A<String> {
    @Override
    String get(String s) {
        return "";
    }
}

public class TestBridge {
    public static void main(String[] args) {
        Class<B> clazz = B.class;
        Method[] methods = clazz.getDeclaredMethods();
        for (int i = 0; i < methods.length; i++) {
            Method m = methods[i];
            System.out.println(getMethodInfo(m) + " is Bridge Method? " + m.isBridge());
        }
    }

    public static String getMethodInfo(Method m){
        StringBuilder sb = new StringBuilder();
        sb.append(m.getReturnType()).append(" ");
        sb.append(m.getName());
        Class[]params = m.getParameterTypes();
        for (int i = 0; i < params.length; i++) {
            sb.append(params[i].getName()).append(" ");
        }
        return sb.toString();
    }
}
```

B 中复写了 A 中的 get 方法，那么在 main 方法中，打印 B 中的方法数，应该就是一个吧，那接下来看一下打印结果：

> [class java.lang.String get] is Bridge Method? false
>
> [class java.lang.Object get] is Bridge Method? true 

会不会感到诧异，怎么会多出一个返回值为 Object 的 get 方法？



这个是 java5 中的泛型所带来的结果了。针对上面的这段代码分析下： 
在 java5 之前，你可以往一个集合里扔任何你想扔的对象。但是从集合中取对象却变得很难。你不知道你下个取到的对象将会是什么具体类型的。因为取出来的对象是 Object 类型的，不知道转成什么类型，所以只能使用所有 Object 的方法了，这样就毫无意义了。所以在 java5 中提供了泛型这一新特性。我们在写代码的时候可以指定集合可以存放对象的类型。然后**将类型检查的事情交给编译器去做**，减少了程序员的工作。 
​    

上面代码中<>中的 T 和 String 就是指定类的参数类型。T 代表一种泛型，告诉编译器，一旦有类指定了 T 这个参数的实际类，那么 get 方法返回的类型也必须为同一个类（当然也可以是这个类的子类；这个也是 java5 中的协变式返回新特性），如果不是，就必须报错提示；将原来的运行时可能出现的错误提前到编译期了。那么，假设你是 java5 编译器的设计者，你会如何来设计让编译器能实现这个特性，同时能保证编译出来的字节码可以在老版本的 jdk 中运行呢？java5 编译器中作了个很巧妙的设计——桥接方法。 



那么编译器是如何编译这个抽象类 A 的呢？

对于 A：

```java
abstract class A<T> {
    abstract T get(T t);
}
```

编译器会直接将其转换为下面的代码：

```java
abstract class A<Object> {
    abstract Object get(Object t);
}
```

上面这个过程称为类型擦除，将泛型类型参数全部替换为 Object。

对于 B 类，它继承了 A 类，指定了 T 参数为 String。如果还按照以前那么编译，那编译的类就是： 

```java
class B extends A {
    String get(String s) {
        return "";
    }
}
```

这样在运行时肯定会报错，因为 B 继承了 A，而 A 又是 asbtract 类，B 还没 overriding A 中 Object get()方法。如何解决这个错误呢？java5 编译器在编译的时候做了些手脚。当编译器发现你指定了类型参数，便会在编译的字节码中添加一个桥接方法。所以代码变成了下面这样：

```java
class B extends A {
    //这个就是编译器添加的方法
    Object get(Object s) {
        return (Object) get((String) s);
    }

    String get(String s) {
        return "";
    }
} 
```

而我们实际在调用 B 的 get 方法时，调用的其实是第二个方法，因为我们的参数传入的是 String。而如果是使用了多态，调用了 A.get,那么调用的将是 B 的第一个 get 方法。

- 第一个 get 方法的描述符是：(Ljava/lang/Object;)Ljava/lang/Object;

  access_flag:0x0001(public)

- 第二个 get 方法的描述符是：(Ljava/lang/String;)Ljava/lang/String;

  access_flag:0x1041(public Synthetic bridge)



## SourceFile

SourceFile 是可选定长属性，只会出现在 ClassFile 结构中，用于指出源文件名。其结构定义如下：

```java
SourceFile_attribute {
  u2 attribute_name_index;
  u4 attribute_length;
  u2 sourcefile_index;
}
```

attribute_length 的值必须是 2。因为这个长度就是下面一个字段 sourcefile_index 的长度，这个索引是常量池的索引，常量池长度是用 u2 来表示的，所以该索引决不能超出 u2 的最大值，因此最大用两字节表示，所以 attribute_length 值固定为 2。

sourcefile_index 是常量池索引，指向`CONSTANT_Utf8_info`常量。而`CONSTANT_Utf8_info`中的字符串就是当前源文件的文件名。



## ConstantValue

ConstantValue 是定长属性，只会出现在 field_info 结构中，用于表示常量表达式的值。其作用是通知虚拟机自动为静态变量赋值。只有被 static 修饰的变量（类变量）才可以使用这项属性。

其结构定义如下：

```java
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
```

下面有三种情况：

- int a1 = 123；
- static int a2 = 123;
- final static int a3 = 123

对于非 static 变量，eg：a1，其赋值是在实例构造器`<init>`方法中完成的。

而对于 static 变量，eg：a2，a3，有两种赋值方式。

1. 在类构造器`<clinit>`方法中。
2. 使用 ConstantValue 属性。

目前 Sun Javac 的选择是：前提都是针对于用 static 修饰的静态变量。如果是用 final static 修饰的话，并且这个变量是基本类型或者 String，那么使用 2 赋值。否则使用 1 赋值。

因此这里的 constantvalue_index 是指向常量池中一个字面量类别（CONSTANT_Integer、CONSTANT_Float、CONSTANT_Long、CONSTANT_Double、CONSTANT_Utf8 五种中的一种）的索引，该常量中保存着变量的值。



## Code

Code 是变长属性，只存在于 method_info 结构中。Code 属性中存放字节码等方法相关信息。其结构定义如下：

```java
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

关于上面的 exception_table，其结构定义如下。

```java
{ 
  	u2 start_pc;
    u2 end_pc;
    u2 handler_pc;
    u2 catch_type;
}
```



java 方法中的代码经过 Javac 编译器处理之后，最终变成**字节码指令**存储在 Code 属性内。Code 属性出现在方法表的属性集合之中，但是并非所有的方法都必须存在 Code 这个属性。譬如接口或者抽象类中的方法就不存在 Code 属性。

接下来对其中的字段做一个介绍：

- attribute_name_index：指向 CONSTANT_Utf8_info 类型常量的索引，这个常量值固定为“Code”，代表了该属性的属性名称。
- attribute_length：代表该属性的长度，包括从 attribute_name_index 开始到 attributes[]数组结束。
- max_stack：代表操作数栈的深度的最大值。在方法执行的任意时刻，操作数栈都不能超过这个深度。
- max_locals：代表了局部变量表所需的存储空间大小。在这里 max_locals 的单位是 Slot，Slot 是虚拟机为局部变量非配内存所使用的最小单位。对于 byte、char、short、int、float、boolean、returnAddress 等长度不超过 32 位的数据，每个局部变量占用一个 Slot，而 double 和 long 这种 64 位的数据则需要两个 Slot 来存放。
- code_length：指示下面的 code 字节码数组的长度。虽然这是一个 u4 类型，理论上最大值可以达到 2^32-1，但是 Java 虚拟机明确规定一个方法中的指令不能超过 65535 条字节码指令，也就是说它实际是使用了 u2 的长度。
- code[code_length]：存放的是 Java 源程序编译后生成的 **字节码指令** ，关于字节码指令，会在[手写JVM系列(9)-指令集](https://zachaxy.github.io/2017/05/13/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-9-%E6%8C%87%E4%BB%A4%E9%9B%86/)一节详细说明。
- exception_table_length：指示下面的异常表数组的长度。
- exception_table[exception_table_length] 关于异常处理，会在[后面的文章]()中进行详细说明。
- attributes_count：指示下面的属性表数组的长度。
- attribute_info attributes[attributes_count]：Code 本身就已经是属性了，在这个属性的字段中还包括一些其它的属性...那么就存在这个表中。



### 关于 max_locals

max_locals 给出局部变量表大小。在这里 max_locals 的单位是 Slot，Slot 是虚拟机为局部变量非配内存所使用的最小单位。对于 byte、char、short、int、float、boolean、returnAddress 等长度不超过 32 位的数据，每个局部变量占用一个 Slot，而 double 和 long 这种 64 位的数据则需要两个 Slot 来存放。



局部变量表中存放的内容

- 方法参数（包括实例方法中隐藏参数 this）
- 显式异常处理器的参数（catch 块所定义的异常）
- 方法体中递归的局部变量

但是这里要注意的是：并非在方法中用到了多少个局部变量，就把这些局部变量所占的 Slot 的数量作为 max_locals 的值，因为局部变量表中的 Slot 是可以重用的，当代码执行超出了某一局部变量的**作用域**之后，这个 Slot 就可以被其它局部变量所使用了，所以 Javac 编译器会根据变量的作用域来分配 Slot 给各个变量使用，然后计算出 max_locals 的大小。



**方法体内部使用的 this 从何而来？**

大家注意到没有，定义在类中的非静态方法内部，可以使用 this 来访问当前的对象内的属性，它的实现方法是在 Javac 编译的时候把 this 添加到每个非静态方法的方法参数中，所以在方法内访问的 this 其实是本方法的参数。我们自己定义一个`void func()`的方法，使用 javap 命令查看其 Code 字节码，会发现这个 func 方法的`Args_size=1`,原因就在这，这个参数就是编译器默认为我们添加进去的`this`



## Exception

这里将的 Exception 属性和 Code 属性是一级的。并不是 Code 属性中的异常属性表。

这里的 Exception 属性的作用是列举方法中通过`throws`关键字后面列举的异常。其结构如下：

```java
Exceptions_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_exceptions;
    u2 exception_index_table[number_of_exceptions];
}
```

这里的 number_of_exceptions 项表示方法可能抛出的异常的数量。后面跟的是一个 exception_index_table，该数组中每个元素的长度均为 u2，因为该数组中保存的是一个指向常量池中 CONSTANT_Class_info 型的常量的索引，代表该异常的类型。
