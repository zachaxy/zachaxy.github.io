---
title: 手写JVM系列(16)反射机制简介
date: 2018-01-18 22:19:01
tags: JVM
---

反射是 java 一个强大的机制，通过反射，我们可以动态地创建类型的实例，将类型绑定到现有对象，或从现有对象中获取类型，然后调用类型的方法或访问其字段和属性。本节主要介绍反射机制中用到的类对象。

<!--more-->

# 理解类对象

我们之前在 JVM 中定义了 Zclass 类，用来表示一个 class 文件在 JVM 中的抽象(注意该 Zclass 和 java/lang/Class 中的做区分，前者是在 JVM 中，后者是在 java 应用程序中的类)。在构造 Zclass 对象时，我们是传入了一个 ClassFile 的对象,然后通过 ClassFile 对象的各个字段，构造了 Zclass 对象。同一个 class 文件会产生唯一的一个 Zclass 对象(前提是用同一个类加载器加载的 class 文件，如果是不同的类加载器，则另当别论，本文暂时不讨论多个类加载器加载同一个 class 文件的情况，所以暂且认为同一个 class 文件和 Zclass 对象是一一对应的)。
```java
public Zclass(ClassFile classFile) {
    accessFlags = classFile.getAccessFlags();
    thisClassName = classFile.getClassName();
    superClassName = classFile.getSuperClassName();
    interfaceNames = classFile.getInterfaceNames();
    runtimeConstantPool = new RuntimeConstantPool(this, classFile.getConstantPool());
    fileds = Zfield.makeFields(this, classFile.getFields());
    methods = Zmethod.makeMethods(this, classFile.getMethods());
    sourceFile = classFile.getSourceFile();
}
```

使用 Zclass 就可以描述一个类，其实我们在 java 中定义一个类，无非就是在定义两类东西，一个是该类中有哪些变量，另一个是该类中有哪些方法。这些都可以在 Zclass 中反映出来。

我们在 JVM 中定义的 Zobject(Zobject 表示的是在 JVM 中一个对象的，要注意其和在 java 应用中 Object 的区别) 中就有一个字段，其指向该对象对应的 Zclass 对象。这个很容易理解，因为 Zobject 中只保存了该对象的成员变量的值，如果要执行该类的方法，还是要去对应的 Zclass 中的 Zmethod 中获取对应的字节码。方法的字节码都是一样的，并不针对单独的对象，同一个类创建的对象，调用其方法时的字节码都是一样的，只不过传入的参数不同而已。因此为了节省 JVM 的内存，没有必要为每个 Zobject 中保存方法的字节码。

这里只是打通了 Zobject 到 Zclass 的单向通道，其实 Zclass 中也需要一个字段指向一个具体的 Zobject，注意该 Zobject 并不是由 Zclass 创建的**实例对象**，而是 Zclass 所代表的 Class 的**类对象**。
我们拿熟悉的 String 类举例子，每个字符串 eg:"hello"都是 String 的一个实例对象，对应 JVM 中的 Zobject。然而 String 所代表的类，也有一个实例对象，该对象的获取方式有两种：
- String.class
- "hello".getClass()

这两种方法获取到的都是同一个对象。

这也解释了为什么 JVM 中每个 Zclass 中都需要一个字段指向该类的类对象(String.class)，而不是类的实例对象 "hello"

接下来我们就在 Zclass 中添加这个字段，让其指向该类的类对象。
```java
public class Zclass {
	...
    Zobject jObject;        // jObject 指向的是该类的元类对象 obj。 eg：String.class 得到的结果
    ...
}
```

定义了该字段，那么该字段具体在上面时候赋值呢？回想一下 Zclass 是何时被创建的，是通过类加载器加载 class 文件是生成的，于是我们也就很自然的想到类对象需要紧跟着在 Zclass 生成之后生成。

# 实现类对象
前面也提到了，为类对象赋值是在 class 文件生成之后，那么我们再次修改类加载器，是其完成类对象的构造。
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

    //为每一个 class 都关联一个元类
    Zclass jlClassClass = map.get("java/lang/Class");
    if (jlClassClass != null) {
        clazz.jObject = jlClassClass.newObject();
        clazz.jObject.extra = clazz;
    }
    return clazz;
}
```
在产生 Zclass 之后，获取`java/lang/Class`的 Zclass——jlClassClass，在利用 jlClassClass 创建 Zobject，此 Zobject 就是之前加载的 Zclass 的类对象，通过对 Zclass 的 jObject 的赋值，那么加载到方法区的类就都有类对象了。


这里还有个问题：类也是对象，而对象又是类的实例，那么在 JVM 中，究竟是先有类还是先有对象呢？

在 JVM 中，我们手动触发的是加载`java/lang/Class`，但是在加载该类的时候，会先加载其父类 `java/lang/Object`，以及 Class 实现的接口类。那么此时 JVM 的方法区中已经有了这几个类的 Zclass 了，接下来再为这几个类创建类变量，其逻辑在下面的 loadBasicClasses 方法中，该方法是在 ZclassLoader 的构造方法中调用的，也就是说 JVM 启动时就触发这些基本类的加载，因为这些类太重要了，所有其它的子类都需要这些类。
```java
private void loadBasicClasses() {
    //经过这一步 load 之后,classMap 中就有 Class 的 Class 了，已经 Object 和 Class 所实现的接口；
    Zclass jlClassClass = loadClass("java/lang/Class");
    //接下来对 classMap 中的每一个 Class 都创建一个 jClass;使用 jlClassClass.NewObject()方法;
    // 通过调用 newObject 方法，为每一个 Class 都创建一个元类对象；这样在使用 String.class 时可以直接获取到；
    for (Map.Entry<String, Zclass> entry : map.entrySet()) {
        Zclass jClass = entry.getValue();
        if (jClass.jObject == null) {
            jClass.jObject = jlClassClass.newObject();
            jClass.jObject.extra = jClass;
        }
    }
}
```

# 基本类型的类
通过前面的介绍，已经完成了类与类对象的关联，但是 void 和基本类型也有对应的类对象，但其只能通过字面值来访问，eg:
- void.class
- int.class

就像之前的数组类一样，基本类型的类并没有 class 文件，其也是由 JVM 在运行期间生成的。那么我们就在 JVM 开启时，也手动创建基本类型的类，并将基本类型的类加载到方法区。

```java
    //加载基本类型的类:void.class;boolean.class;byte.class
private void loadPrimitiveClasses() {
    for (Map.Entry<String, String> entry : ClassNameHelper.primitiveTypes.entrySet()) {
        String className = entry.getKey();
        loadPrimitiveClass(className);
    }
}

//加载基本类型,和数组类似,也没有对应的 class 文件,只能在运行时创建;基本类型:无超类,也没有实现任何接口
/* 针对基本类型的三点说明：
1. void 和基本类型的类型名字就是：void，int，float 等
2. 基本类型的类没有超类，也没有实现任何接口
3. 非基本类型的类对象是通过 ldc 指令加载到操作数栈中的
*/
private void loadPrimitiveClass(String className) {
    Zclass clazz = new Zclass(AccessFlag.ACC_PUBLIC, className, this, true,
            null,
            new Zclass[]{});
    clazz.jObject = map.get("java/lang/Class").newObject();
    clazz.jObject.extra = clazz;
    map.put(className, clazz);
}


public static HashMap<String, String> primitiveTypes;

static {
    primitiveTypes = new HashMap<String, String>();
    primitiveTypes.put("void", "V");
    primitiveTypes.put("boolean", "Z");
    primitiveTypes.put("byte", "B");
    primitiveTypes.put("short", "S");
    primitiveTypes.put("int", "I");
    primitiveTypes.put("long", "J");
    primitiveTypes.put("char", "C");
    primitiveTypes.put("float", "F");
    primitiveTypes.put("double", "D");
}
```


对于非基本类型的类对象，其获取是通过 ldc 指令获取到的，
```
static void fClazz() {
    Class c = String.class;
}
//对应的字节码：
0 ldc #4 <java/lang/String>
2 astore_0
3 return
```

回想一下，之前介绍的 ldc 指令的功能，是用来加载基本类型，字符串字面值的，所以现在需要扩展 ldc 指令，根据 ldc 操作数得到的运行时常量池中的常量，如果是 ClassRef 类型的，那么就将其 Zclass 的 jObject 对象放到操作数栈
```java
public class LDC extends Index8Instruction {
    @Override
    public void execute(Zframe frame) {
        OperandStack operandStack = frame.getOperandStack();
        Zclass clazz = frame.getMethod().getClazz();
        RuntimeConstantInfo runtimeConstant = clazz.getRuntimeConstantPool().getRuntimeConstant(index);
        switch (runtimeConstant.getType()) {
            case ConstantInfo.CONSTANT_Integer:
                operandStack.pushInt((Integer) runtimeConstant.getValue());
                break;
        ......
            case ConstantInfo.CONSTANT_String:
                Zobject internedStr = StringPool.jString(clazz.getLoader(), (String) runtimeConstant.getValue());
                operandStack.pushRef(internedStr);
                break;
            case ConstantInfo.CONSTANT_Class:
                ClassRef classRef = (ClassRef) runtimeConstant.getValue();
                Zobject jObject = classRef.resolvedClass().getjObject();
                operandStack.pushRef(jObject);
                break;
            // case MethodType, MethodHandle //Java7 中的特性，不在本虚拟机范围内
            default:
                break;
        }
    }
}
```



同时，基本类型的类对象，在 java 代码中看起来是通过字面量获取的，但是通过查看其编译后的 class 文件，发现并不是由 ldc 加载到内存的，而是通过 getstatic 指令。因为每个基本类型都有对应的包装类，包装类中都有一个静态常量：TYPE，该字段中保存的就是基本类型的类，我们查看 Integer.java 类
```java
public final class Integer extends Number implements Comparable<Integer> {
	 public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
}
```

而 getPrimitiveClass 方法是一个 native 的方法，关于 native 的方法的实现，请参考下一节的讲解。