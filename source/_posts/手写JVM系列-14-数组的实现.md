---
title: 手写JVM系列(14)-数组的实现
date: 2018-01-12 19:18:25
tags: JVM
---

前面介绍了[类加载器的实现](https://zachaxy.github.io/2018/01/04/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-12-%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E7%9A%84%E5%AE%9E%E7%8E%B0/)，目前我们已经可以加载普通的类，并创建该类的对象了，但是目前我们的 JVM 还不支持数组。数组类虽然也是类，但是其加载方式和之前所讲的通过 class 文件的形式加载是完全不同的，本节将介绍 JVM 中数组的实现，并完善我们的类加载器，使其可以加载数组。

<!--more-->

# 数组概述

- 基本数组类型：数组中的元素类型是基本类型
- 引用数组类型：数组中的元素类型是引用类型



数组类也是类，其父类同样也是 Object，但是其又和普通的类有一些差异：
- 普通的类从 class 文件中加载,而数组类则是 JVM 在运行时生成的.
- 创建普通对象和创建数组所用的指令是不同的;前者用 new,后者用 newarray 等指令。
- 访问对象的成员变量和数组的元素不同，前者使用的是 getfield 指令，后者使用的是 aload 指令。
- 数组的类名和其它类也不同，数组的类名是用左括号+数组元素的类型描述符。同时，数组的类名和数组的类型描述符是一样的。eg：int[]的类名为：[I；int[][] 的类名为：[[I；String[][] 的类名为： [[java/lang/String；



# 数组的创建
普通类和数组类的创建方式不同,但是其归根结底还是 Object 的子类。所以要在之前的 Zclass 中添加额外的针对创建数组 class 的方法。同样由数组的 class 创建的数组 Object 和普通 Object 不同。所以还要在 Object 中添加额外的方法来创建数组 Object。可是不管数组类和普通类有什么区别，整体的流程都是一样的，凡是创建对象,都是使用 newXXX 的指令,通过对应的指令先创建其对应的 class,然后用该 class 来创建 object。
接下来依次从加载数组类，然后根据数组类创建数组对象的流程，讲解数组的创建过程。


## 加载数组类
加载数组类和加载普通类的时机都是一样的，如果在 java 代码中显式的 new 了一个数组，那么势必会触发对应数组类的加载，而创建数组类的指令有三个：
- newarray：用于创建基本类型的一维数组，其操作数为一字节，该字节用来表示对应的基本类型
- anewarray：用于创建引用类型的一维数组，其操作数为 2 字节，指向运行时常量池的一个类引用，该引用的类型就是引用类型数组中的元素类型
- multianewarray：用于创建多维数组，其操作数为 3 字节，前两个字节同 anewarray，后一个字节用来表示多维数组的纬度，由此也可以看出我们在 Java 中创建多维数组最大纬度为 255。

这里以 newarray 指令为例，查看其数组类加载的过程：
前面也说了 newarray 所创建的基本类型在指令后的字节中，JVM 为基本类型设置了对应的值，我们可以根据值来加载对应的类
```java
private final int AT_BOOLEAN = 4;
private final int AT_CHAR = 5;
private final int AT_FLOAT = 6;
private final int AT_DOUBLE = 7;
private final int AT_BYTE = 8;
private final int AT_SHORT = 9;
private final int AT_INT = 10;
private final int AT_LONG = 11;


switch (this.index) {
    case AT_BOOLEAN:
        return loader.loadClass("[Z");
    case AT_BYTE:
        return loader.loadClass("[B");
    case AT_CHAR:
        return loader.loadClass("[C");
    case AT_SHORT:
        return loader.loadClass("[S");
    case AT_INT:
        return loader.loadClass("[I");
    case AT_LONG:
        return loader.loadClass("[J");
    case AT_FLOAT:
        return loader.loadClass("[F");
    case AT_DOUBLE:
        return loader.loadClass("[D");
    default:
        throw new RuntimeException("Invalid atype!");
}
```

然后用类加载器去加载对应的类。
```java
public Zclass loadClass(String name) {
    if (map.containsKey(name)) {
        return map.get(name);
    }

    Zclass clazz;
    if (name.charAt(0) == '[') {
        clazz = loadArrayClass(name);
    } else {
        clazz = loadNonArrayClass(name);
    }
    ...
    return clazz;
}
```

## 创建数组类
在加载对应的基本类型数组的时候，并不是从字节码文件中读取，然后转换为对应的 Class 对象，而是根据基本数组类型，直接创建对应的 class 对象。
```java
private Zclass loadArrayClass(String name) {
    Zclass clazz = new Zclass(AccessFlag.ACC_PUBLIC, name, this, true,
            loadClass("java/lang/Object"),
            new Zclass[]{loadClass("java/lang/Cloneable"), loadClass("java/io/Serializable")});
    map.put(name, clazz);
    return clazz;
}
```

修改 Zclass ，我们之前创建 class 对象都是根据 class 文件，通过传入一个 ClassFile 对象构造 Class 对象的，而对于数组类型，我们为其添加一个构造方法，用来直接创建数组类型
```java
public Zclass(int accessFlags, String thisClassName, ZclassLoader loader,
              boolean initStarted, Zclass superClass, Zclass[] interfaces) {
    this.accessFlags = accessFlags;
    this.thisClassName = thisClassName;
    this.loader = loader;
    this.initStarted = initStarted;
    this.superClass = superClass;
    this.interfaces = interfaces;
}
```

## 创建数组对象
上面创建的数组类型的 class 对象，那么接下来就该利用该 class 对象创建数组对象了，在 Zclass.java 中添加 newArray 方法。
```java
public Zobject newArray(int count) {
    if (!isArray()) {
        throw new RuntimeException("Not array class: " + thisClassName);
    }
    switch (thisClassName) {
        case "[Z":
            return new Zobject(this, new byte[count], null);
        case "[B":
            return new Zobject(this, new byte[count], null);
        case "[C":
            return new Zobject(this, new char[count], null);
        case "[S":
            return new Zobject(this, new short[count], null);
        case "[I":
            return new Zobject(this, new int[count], null);
        case "[J":
            return new Zobject(this, new long[count], null);
        case "[F":
            return new Zobject(this, new float[count], null);
        case "[D":
            return new Zobject(this, new double[count], null);
        default:
            return new Zobject(this, new Zobject[count], null);
    }
}
```
该方法需要得到数组的大小，在 java 中所创建的数组时是需要确定其具体大小的，该值保存在执行创建数组的 frame 中的操作数栈中。同时在创建对象时，根据 thisClassName 确定具体的类型，然后创建具体的 obj。

同时注意到我们在创建 obj 对象时，是将创建好的数组赋值给了 Zobject 的 data 成员。该 data 成员在还没有介绍数组之前，是一个 slot[] 类型,用来保存非数组对象中的非静态成员变量,包含父类+ 自己的，现在引入了数组类型，需要将 Zobject 中的 data 类型做一个修改，这里直接改为 Object 类型，就可以做到既能盛放 slot，又能盛放数组对象。只不过在对普通对象和数组对象做元素存取的时候还要再进行强制类型转换，才能得到正确的值。其实我们可以这样理解，数组对象也是一个对象，只不过其内部的成员变量就是其数组的所有元素而已。


## 创建基本类型数组的指令 newarray 执行过程
其实前面已经将基本类型数组对象的创建过程进行了一次描述了，这里只是贴出该指令执行的代码，作为回顾。
```java
public class NEW_ARRAY extends Index8Instruction {
    //Array Type  atype
    private final int AT_BOOLEAN = 4;
    private final int AT_CHAR = 5;
    private final int AT_FLOAT = 6;
    private final int AT_DOUBLE = 7;
    private final int AT_BYTE = 8;
    private final int AT_SHORT = 9;
    private final int AT_INT = 10;
    private final int AT_LONG = 11;

    @Override
    public void execute(Zframe frame) {
        OperandStack operandStack = frame.getOperandStack();
        //从栈中获取数组的大小
        int count = operandStack.popInt();
        if (count < 0) {
            throw new NegativeArraySizeException("" + count);
        }
        ZclassLoader loader = frame.getMethod().getClazz().getLoader();
        Zclass arrClazz = getPrimitiveArrayClass(loader);
        Zobject arr = arrClazz.newArray(count);
        operandStack.pushRef(arr);
    }

    //获取基本类型数组的 class;如果没有加载过,需要加载进 JVM
    private Zclass getPrimitiveArrayClass(ZclassLoader loader) {
        //从字节码中获取到的 index 表明的是哪种类型的数组
        switch (this.index) {
            case AT_BOOLEAN:
                return loader.loadClass("[Z");
            case AT_BYTE:
                return loader.loadClass("[B");
            case AT_CHAR:
                return loader.loadClass("[C");
            case AT_SHORT:
                return loader.loadClass("[S");
            case AT_INT:
                return loader.loadClass("[I");
            case AT_LONG:
                return loader.loadClass("[J");
            case AT_FLOAT:
                return loader.loadClass("[F");
            case AT_DOUBLE:
                return loader.loadClass("[D");
            default:
                throw new RuntimeException("Invalid atype!");
        }
    }
}
```


# 完善数组相关的其它指令
前面介绍了创建基本类型的数组指令 newarray，其实还有其它和数组相关的指令，这里做一个概述，具体实现请参考[instructions/references 包下的源码](https://github.com/zachaxy/JVM/tree/master/Java/src/instructions/references)

## anewarray
创建引用类型的一维数组。其过程和创建基本类型的一维数组是类似的，只不过该数组的元素是引用类型，所以我们要先获取到数组元素的引用类型。由 anewarray 指令后的操作数获取，其操作数为 2 字节，指向运行时常量池的一个类引用，该引用的类型就是引用类型数组中的元素类型。我们在获取到该类引用之后，先将其转换为直接引用(若还为加载过该类，则需要先加载到方法区)。接下来根据类型名，创建其一维数组名，eg：数组元素为 String，那么其一维数组类型名就是：[java/lang/String；接下来创建引用类型的一维数组的过程就和创建基本类型的一维数组过程就一样了。


## arraylength
获取数组的长度，在 java 中也是很常用的，其实现也很简单，我们之间在创建数组对象的时候，将数组放到了 Zobject 的 data 变量中，我们可以通过 data 的 class 来获取原本数组类型名，根据类型名将 data 进行相应的类型转换，得到具体的数组之后，在获取其长度。

## <t>aload 和 <t>astore
分别用来读取和写入数组元素，功能和之前介绍的 load，store 指令相似，只不过需要从操作数栈获取数组的索引，然后在对数组中对应的操作。比较简单，直接看代码，这里不再赘述了。
其实现分别在`instructions.loads.loadxarr`包下和`instructions.stores.storexarr`包下。

## multianewarray
这里以`new int[3][4][5]`为例，讲解多维数组的创建过程，具体实现过程，请参考[MULTI_ANEW_ARRAY.java](https://github.com/zachaxy/JVM/tree/master/Java/src/instructions/references/MULTI_ANEW_ARRAY.java)
代码 `int[][][] arr = new int[3][4][5];` 
其产生的字节码为：
```
iconst_3
iconst_4
iconst_5
multianewarray #5 ([[[I,3)
```
首先将三维数组各个维度压入操作数栈，栈顶向下一次为： 5，4，3，然后执行 multianewarray 指令，其操作数有两个，第一个 index 表示运行时常量池的类符号引用，其类名为[[[I
接着获取第二个操作数 3，表明这是一个三维数组。
接下来开始执行 multianewarray 指令：首先将获取到的类符号引用转为直接引用：转换依然是用 classloader，因为是数组类，所以不用从 class 文件中读取字节流，而是直接创建一个 class，该 class 将类名指定为 [[[I，这一创建 Class 对象的过程和创建一维数组是一样的；接下来依次从操作数栈中弹出三个整数，表示该多维数组每一维的大小；然后开始创建该多维数组的对象

多维数组对象的创建过程：此时拿到的类名是：[[[I，各个维的大小是 3，4，5；
首先利用数组类[[[I,创建第一维 arr1 ，大小为 3，（多维数组对外表现的就是一维数组，只不过该数组中的元素依然是数组。）
接下来创建 arr1 中的每一个元素，其元素也是数组，我们称之为第 2 维，arr2（arr1 中的三个元素都是 arr2）
arr2 此时的类型为 [[I，依然需要用 classloader 进行加载，然后创建；
接下来创建 arr2 中的每一个元素，其元素还是数组，我们称之为第 3 维，arr3（arr2 中的三个元素都是 arr3）
arr3 此时的类型为 [I,依然需要用 classloader 进行加载，然后创建；
最终将创建好的 arr1，压入操作数栈，结束！


# 测试
本节的测试代码在(TestNewArray09.java 中)[https://github.com/zachaxy/JVM/blob/master/Java/src/test/TestNewArray09.java]。
