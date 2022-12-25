---
title: 手写JVM系列(18)异常处理机制
date: 2018-01-20 16:58:41
tags: JVM
---

前面介绍了方法的调用过程，简单的方法可以在我们自己写的 JVM 中运行起来了，但是方法的执行过程并不完整，因为方法可能会抛出异常，我们暂时还无法处理异常。本节依然通过示例来讲解异常的处理机制。进一步完善我们的 JVM。核心是 [athrow 指令的实现](https://github.com/zachaxy/JVM/blob/master/Java/src/instructions/references/ATHROW.java)

<!--more-->

# 本节异常处理机制所用的示例
本节所讲解的异常处理机制是基于以下面的例子做说明的，有两个方法，(1)func 方法直接抛出了一个异常；(2)test 方法调用了 func 方法，同时试图用 try-catch 来捕获异常，但是其所捕获的异常并非 func 抛出的异常，也就是说 test 方法处理不了，最终交由 JVM 来处理，而 JVM 默认的处理方式则是打印虚拟机堆栈信息。
```java
public static void test() {
    int i = 100;
    try {
        func();
    } catch (NumberFormatException e) {
        i = 200;
    }
}

public static void func() {
    throw new IndexOutOfBoundsException();
}
```

test 方法的字节码：
```
 0 bipush 100
 2 istore_0
 3 invokestatic #60 <test/TestException12.func>
 6 goto 14 (+8)
 9 astore_1
10 sipush 200
13 istore_0
14 return
```

# 直观的感受 athrow 命令
由上面的 func 方法，其方法体直接通过 throw 关键字抛出了一个异常，那么查看 func 方法对应的字节码:
```
0 new #62 <java/lang/IndexOutOfBoundsException>
3 dup
4 invokespecial #63 <java/lang/IndexOutOfBoundsException.<init>>
7 athrow
```
前面的三个指令是在创建 IndexOutOfBoundsException 异常的一个实例，并执行其 init 方法。
接下来就是一个 athrow 命令，将上面创建的异常抛出。这就是 athrow 指令出现的场景。现在只是只管的感受一下 athrow 指令，暂时先不去实现它。

# 异常处理表
func 方法内部抛出了异常，那么接下来需要另一个方法 test 来调用 func 方法。在 test 方法中，我们 catch 了一个异常 NumberFormatException，但是该异常的类型并不是 func 方法抛出的异常，因此是不会执行对应的 catch 块的。我们通过定义一个变量 i 来标示，如果执行了 catch 块，那么该变量 i 会变为 200，否则还是初值 100；

如果在方法中使用了 try-catch 语句，那么在该方法的 code 属性中就会有一个异常处理表。这里之所以叫表，是因为我们可能会 catch 多个异常，每 catch 一个异常，就会产生一条表项。
在上面的 test 方法中就有异常表，因为我们这里只有一个 catch，所以异常表只有一条表项
```
{
    start:3
    end:6
    hadlerPC:9
    catchType: #5 (java/lang/NumberFormatException)
}
```

JVM 中定义的异常处理表项有四个字段：均为整数
- startPc
- endPc
- handlerPc
- catchType

其含义如下：
[startPc,endPc) 一个区间:锁定一个指令区间,该区间可能抛出某种异常。test 的异常表表示在字节码 3 到字节码 6(不包含 6)之间，可能会抛出异常，而指令 3 正是调用了抛出异常的 func 方法。
handlerPc:负责处理异常的 catch 块的其实位置，一旦发现某个 catch 可以处理该异常，那么就将 nextPc 改为 handlerPc。
catchType:指向运行时常量池的一个索引,通过该索引可以得到一个异常的类 xxxException 的对象，也就是我们在代码中显式 catch 的异常的类型。

# 解析异常处理表
和之前的常量池类似，异常处理表也是保存 class 文件中，我们在读取 class 文件的时候，一样要对其进行转换。
首先看一下其在 ClassFile 中的表现形式:
```java
public static class ExceptionTableEntry {
    int startPc;        //可能排除异常的代码块的起始字节码（包括）
    int endPc;          //可能排除异常的代码块的终止字节码（不包括）
    int handlerPc;      //负责处理异常的 catch 块的其实位置
    int catchType;      //指向运行时常量池的一个索引，解析后可以得到一个异常类
}
```

其在 Zclass 中的表现：
```java
class ExceptionHandler {
    int startPc;
    int endPc;
    int handlerPc;
    ClassRef catchType;
}
```

对比发现，具体的转换是将 catchType 运行时常量池的常量直接转换为对应的异常类的引用，其转换过程是在 ClassFile->Zclass 的转换过程中，具体对应到方法的转换过程。
```java
public Zclass(ClassFile classFile) {
    ...
    runtimeConstantPool = new RuntimeConstantPool(this, classFile.getConstantPool());
    fileds = Zfield.makeFields(this, classFile.getFields());
    methods = Zmethod.makeMethods(this, classFile.getMethods());
    ...
}
```



# athorw 命令的实现(异常处理流程)
再次贴出 test 方法的代码
```java
public static void test() {
    int i = 100;
    try {
        func();
    } catch (NumberFormatException e) {
        i = 200;
    }
}
```

test 对应的栈帧为 frame1，当其内部执行 func 方法时，会为其创建栈帧 frame2，在 frame2 中执行 func 方法，在 func 方法中，首先创建了一个异常——IndexOutOfBoundsException 的对象，该对象放在 frame2 的操作数栈顶。然后遇到了 athrow 命令，该指令将 frame2 操作数栈顶的异常对象 ex pop 出来，然后从当前 frame 开始遍历虚拟机栈。我们知道当前 JVM 的栈为 frame1 -> frame2。那么先从 frame2 开始，查找当前方法的异常处理表，当然我们一般在写代码的时候不会在手动写了 throw 异常之后，立刻在当前方法就去 catch 它，我们一般的做法都是在 func 方法的声明处显示的 throws 对应的异常，等待调用 func 的上层方法来出来，但是语法也是允许自己来 catch 的，所以先从 frame2 查找，查找不到，那么将 frame2 从虚拟机栈弹出，然后再从当前栈顶的 frame1 中查找是否可以找到异常处理项，就这样一直遍历下去。

这样的遍历会产生两个结果：
1. 在某个 frame 中找到了可以处理该异常的项，那么首先到该 frame 的操作数栈清空，然后把异常对象压入当前操作数栈，接着执行该 frame 对应的 catch 块语句。
2. 遍历了整个虚拟机栈都没有找到可以处理的异常项，因此交由 JVM 来处理，而 JVM 的处理方式就是打印虚拟机栈信息(从产生异常的方法 func 对应的 frame，依次到 main 方法对应的 frame)

athrow 指令的处理流程：
```java
Zobject ex = frame.getOperandStack().popRef();

//获取当前 thread,是想从 thread 中获取所有的 frame(一个 frame 就是一个方法调用过程的抽象)
Zthread thread = frame.getThread();
if (!findAndGotoExceptionHandler(thread, ex)) {
    //所有的方法调用栈均不能处理该异常了,那么交由 JVM 来处理;
    handleUncaughtException(thread, ex);
}
```

上面多次提到了在当前 frame 寻找对应的异常处理项，如何寻找，这就用到了 findAndGotoExceptionHandler 方法，具体流程如下：
```java
 //从当前帧开始,遍历 Java 虚拟机栈,查找方法的异常处理表
private boolean findAndGotoExceptionHandler(Zthread thread, Zobject ex) {
    while (true) {
        Zframe frame = thread.getCurrentFrame();
        //正常情况下,获取一条指令后,bytereader 中的 pc 是要+1,指向下一条指令的地址
        // 但是 athrow 指令比较特殊,因为现在已经抛出异常了,不能继续向下执行了,所以要回退;
        int pc = frame.getNextPC() - 1;

        //此时 pc 指向的是 throw 异常的那一行;因为接下来就要在当前方法的异常处理表中寻找可以处理当前 pc 指向的指令所抛出的异常了
        int handlerPC = frame.getMethod().findExceptionHandler(ex.getClazz(), pc);

        //当前方法能处理自己抛出的异常
        if (handlerPC > 0) {
            OperandStack operandStack = frame.getOperandStack();
            operandStack.clear();//清空操作数栈
            operandStack.pushRef(ex);//将抛出的异常放进去
            frame.setNextPC(handlerPC);//指令跳转到对应的 catch 块中
            return true;
        }

        //能走到这一步,说明当前方法不能处理当前方法抛出的异常,需要回到调用该方法的帧 frame
        thread.popFrame();

        //整个调用栈都无法处理异常，交由 JVM 来处理吧；JVM 处理的方法就是下面的 handleUncaughtException
        if (thread.isStackEmpty()) {
            break;
        }
    }
    return false;
}
```

超找异常处理项，最终是在 Method 的 ExceptionTable 对象中
```java
//返回能解决当前 Exception 的 handler=>多个 catch 块,决定用哪个
public ExceptionHandler findExceptionHandler(Zclass exClazz, int pc) {
    for (int i = 0; i < exceptionTable.length; i++) {
        ExceptionHandler handler = exceptionTable[i];
        if (pc >= handler.startPc && pc < handler.endPc) {
            // catch all
            if (handler.catchType == null) {
                return handler;
            }
            // 如果 catch 的异常是实际抛出的异常的父类，也可以捕获
            Zclass catchClazz = handler.catchType.resolvedClass();
            if (catchClazz == exClazz || catchClazz.isSuperClassOf(exClazz)) {
                return handler;
            }
        }
    }
    return null;
}
```

上面能找到对应的异常处理项，那么整个程序还可以正常运行，但如果没有找到的话，就要交给 JVM 来处理的，此时也意味着当前的虚拟机栈已经被清空了，解释器也停止执行了，JVM 将整个调用链打印出来，程序结束。

# Java 虚拟机栈信息
```java
public class NStackTraceElement {
    String fileName;//类所在的 java 文件
    String className;//声明方法的类名
    String methodName;//调用方法名
    int lineNumber;//出现 exception 的行号
}
```
NStackTraceElement 代表每个 frame 中调用抛出异常方法的具体消息。

上面实现了 ATHROW 命令，但是目前程序还是无法运行，因为异常的父类 Exception 的构造方法中会调用一个 native 的方法——fillInStackTrace，因此还要在 native 方法中注册 fillInStackTrace 方法，同时在该方法中进行虚拟机调用栈的收集工作。
具体的代码请参考[Nthrowable.java](https://github.com/zachaxy/JVM/blob/master/Java/src/znative/java/lang/Nthrowable)，注释已经写的很清楚，相信不会有太大的阅读困难。


# 异常的测试用例
同样，这里提供了一个单独的[测试用例](https://github.com/zachaxy/JVM/blob/master/Java/src/test/TestException12.java)来测试异常，我们的 test 方法并没有不会 func 方法抛出的异常，这一点从最终打印 test 的本地变量表就可以发现，i 的值为 100，没有走到 catch 语句中。同时程序也打印出了虚拟机堆栈信息，具体打印信息如下：
```
the same as testInterpreter06!
java -cp /Users/zachaxy/TestClassFiles  TestException12
current instruction: 0: BIPUSH
current instruction: 2: ISTORE_0
current instruction: 3: INVOKE_STATIC
current instruction: 3: INVOKE_STATIC
test.TestException12.f1(TestException12.java:82)
test.TestException12.test(TestException12.java:75)
100
```