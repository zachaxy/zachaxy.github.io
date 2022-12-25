---
title: 手写JVM系列(8)-运行时数据
date: 2017-05-12 22:47:02
tags: JVM
---

> 前面介绍了如何搜索和解析 class 文件，如何解析 class 文件，那么接下来就要开始实现 JVM 了，这一节我们讨论并初步实现运行时数据区（run-time data area），为接下来编写字节码解释器做准备。本节所有的[代码](https://github.com/zachaxy/JVM)均位于 runtimedata 包下。



# 运行时数据区概述

运行时数据区可以分为两类：

- 多线程共享:在 Java 虚拟机启动时创建好，在 Java 虚拟机退出时销毁。
  - 类数据，其存放在方法区(逻辑上来说,方法区也是堆中的一部分)
    - 字段
    - 方法信息
    - 方法字节码
    - 运行时常量池
  - 类实例(对象)=>堆(堆由垃圾收集器定期清理，所以程序员不需要关心对象空间的释放)
- 线程私有:创建线程时才创建，线程退出时销毁。其主要作用是辅助执行 Java 字节码
  - pc 寄存器：存放当前正在执行的 Java 虚拟机指令的地址，如果当前执行的是 native 方法，则 pc 的值 JVM 没有明确定义。
  - Java 虚拟机栈
    - 栈帧
      - 局部变量表
      - 操作数



 在任一时刻，某一线程肯定是在执行某个方法。这个方法叫作该线程的**当前方法**；

执行该方法的帧叫作线程的**当前帧**；

声明该方法的类叫作**当前类**。

如果当前方法是**Java 方法**，则 pc 寄存器中存放当前正在执行的 Java 虚拟机指令的地址，否则，当前方法是本地方法，pc 寄存器中的值没有明确定义。

![运行时数据区示意图](https://geosmart.github.io/2016/03/07/JVM%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%8C%EF%BC%89Java%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F%E4%B8%8E%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%BC%82%E5%B8%B8/Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE%E5%8C%BA.jpg)

堆可以是连续空间，也可以不连续。堆的大小可以固定，也可以在运行时按需扩展。

Java 命令提供了-Xms 和-Xmx 两个非标准选项，用来调整堆的初始大小和最大大小。如果没有进行指定,那么使用默认大小;

虚拟机实现者可以使用任何垃圾回收算法管理堆，甚至完全不进行垃圾收集也是可以的。



# 数据类型

Java 虚拟机可以操作两类数据：

- 基本类型:变量中存放数据本身。
- 引用类型:变量中存放的是对象引用，真正的对象数据是在堆里分配的。

这里所说的变量包括 类变量（静态字段）、实例变量（非静态字段）、数组元素、方法的参数和局部变量等

引用类型可以进一步分为 3 种：

- 类类型:指向类实例
- 接口类型:指向实现了该接口的类或数组实例
- 数组类型类:指向数组实例 


引用类型有一个特殊的值——null，表示该引用不指向任何对象。



**操作数栈和局部变量表只存放数据的值,并不记录数据类型。具体操作的是什么类型的数据,反映在操作符上,eg:iadd:就是对 int 值进行加法操作.**

关于字节码和指令集的具体讲解在[手写JVM系列(9)-指令集](https://zachaxy.github.io/2017/05/13/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-9-%E6%8C%87%E4%BB%A4%E9%9B%86/)中介绍。

基于 Java 虚拟机只能操作上面两种数据，并且在[属性表](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/#Code)的 Code 属性一节中介绍过，局部变量是存放在 Slot 中的，所以这里定义一个 Slot 类，里面存放上面所述的两种变量：基本类型和引用类型。

```java
public class Slot {
    public int num;
    public Zobject ref;

    public Slot() {}
}
```

对于基本类型，使用定义的 num 变量，对于引用类型，使用 Zobject 变量，因为现在还没有涉及到实现类和对象的实现，所以这里自己定义了一个类 Zobject 来代表引用。等到后面章节中，会具体讲解实现类和对象的实现方式。

```java
public class Zobject {
    // TODO: 暂时为空实现  
}
```



# 实现线程私有的运行时数据区

## 线程

既然是要实现线程私有的运行时数据区，那么就要先实现线程类，这里定义 Zthread 类代表线程。

```java
public class Zthread {
    int pc;
    Zstack stack; //Stack 结构体（Java 虚拟机栈）的引用;

    public Zthread() {
        //默认栈的大小是 1024,也就是说可以存放 1024 个栈帧
        stack = new Zstack(1024);
    }

    public int getPc() {
        return pc;
    }

    public void setPc(int pc) {
        this.pc = pc;
    }

    public void pushFrame(Zframe frame) {
        stack.push(frame);
    }

    public Zframe popFrame() {
        return stack.pop();
    }

    public Zframe getCurrentFrame() {
        return stack.top();
    }

    public Zframe createFrame(int maxLocals, int maxStack) {
        return new Zframe(this, maxLocals, maxStack);
    }
}
```

这里只关注其中的成员变量。我们知道线程中私有的数据包括：PC 和虚拟机栈。PC 使用一个整数保存，虚拟机栈使用了我们自定义的一个 Zstack 类，下面是其具体实现。



## Java 虚拟机栈

Java 虚拟机规范对 Java 虚拟机栈的约束非常宽松。虚拟机栈可以是连续的空间，也可以不连续可以是固定大小，也可以在运行时动态扩展。Java 提供了-Xss 选项来设置 Java 虚拟机栈大小。

如果 Java 虚拟机栈有大小限制，且执行线程所需的栈空间超出了这个限制，会导致 StackOverflowError 异常抛出。

如果 Java 虚拟机栈可以动态扩展，但是内存已经耗尽，会导致 OutOfMemoryError 异常抛出。

这里我们用经典的链表（linked list）数据结构来实现 Java 虚拟机栈，这样栈就可以按需使用内存空间，而且弹出的帧也可被及时回收。

```java
public class Zstack {
    int maxSize;    //虚拟机栈中所包含栈帧的最大容量
    int size;       //当前虚拟机栈中包含帧的数量
    private Zframe _top; //栈顶的帧

    public Zstack(int maxSize) {
        this.maxSize = maxSize;
    }


    //新添加一个栈帧,将这个栈帧设置为 top,当然如果当前栈之前有元素,那么将要 push 进的 frame 的 lower 是指为之前的 top,当前 frame 变为 top;
    void push(Zframe frame) {
        if (size > maxSize) {
            //throw new RuntimeException("java.lang.StackOverflowError");
            //如果栈已经满了，按照 Java 虚拟机规范，应该抛出 StackOverflowError 异常
            throw new StackOverflowError();
        }
        if (_top != null) {
            frame.lower = _top; // frame 中保存前一个帧的引用,使得当前帧被 push 的时,前一个帧顶上去;
        }

        _top = frame;
        size++;
    }

    Zframe pop() {
        if (_top == null) {
            throw new EmptyStackException();
        }
        Zframe tmp = _top;
        _top = tmp.lower;
        tmp.lower = null;  //tmp 是带 pop 出的栈帧,既然要 pop 出来,那么将其 lower 设置为 null,不在持有栈中的帧,避免内存泄露;
        size--;
        return tmp;
    }

    Zframe top() {
        if (_top == null) {
            throw new EmptyStackException();
        }
        return _top;
    }
}
```

其中：maxSize 字段保存栈的容量（最多可以容纳多少帧），size 字段保存栈的当前大小，_top 字段保存栈顶指针。

push（）方法把帧推入栈顶，如果栈已经满了，按照 Java 虚拟机规范，应该抛出 StackOverflowError 异常。

pop（）方法把栈顶帧弹出，如果此时栈是空的，肯定是我们的虚拟机有 bug，这里我们手动抛出了一个 EmptyStackException 异常。

top（）方法只是返回栈顶帧，但并不弹出。



## 栈帧

上面的虚拟机栈是用单向链表实现的，栈中每一个元素都是栈帧，这里我们定义一个 Zframe 类，具体实现如下：

```java
public class Zframe {
    Zframe lower;       //当前帧的 前一帧的引用;相当于单向链表的前一个指针
    LocalVars localVars;    //局部变量表的引用;
    OperandStack operandStack;  //操作数栈的引用;
    Zthread thread;
    int nextPC;

    // TODO: 2017/5/4 0004
    public Zframe(Zthread thread, int maxLocals, int maxStack) {
        this.thread = thread;
        localVars = new LocalVars(maxLocals);
        operandStack = new OperandStack(maxStack);
    }

    public LocalVars getLocalVars() {
        return localVars;
    }

    public OperandStack getOperandStack() {
        return operandStack;
    }

    public Zthread getThread() {
        return thread;
    }

    public int getNextPC() {
        return nextPC;
    }

    public void setNextPC(int nextPC) {
        this.nextPC = nextPC;
    }
}
```

lower 字段用来实现链表数据结构，保存的是栈中前一个栈帧的引用，相当于链表中的 next 指针，这样当栈顶的帧出栈时，栈顶帧把其持有的 lower 帧设置为当前栈顶帧。

localVars 字段保存局部变量表引用。

operandStack 字段保存操作数栈引用。

在构造方法中传入了局部变量表大小和操作数栈深度，这两个值是由编译器预先计算好的，存储在 class 文件 method_info 结构的 Code 属性中。



## 局部变量表

局部变量表是按索引访问的，那么我们自然也是用数组来实现。根据 Java 虚拟机规范，这个数组的每个元素至少可以容纳一个 int 或引用值，两个连续的元素可以容纳一个 long 或 double 值。这就是我们之前讲的属性表中 Code 属性中的[Slot](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/#Code)。

```java
public class Slot {
    public int num;
    public Zobject ref;

    public Slot() {}
}
```

操作局部变量表和操作数栈的指令都是隐含类型信息的。下面给 LocalVars 类型定义一些方法，用来存取不同类型的变量。

- int 变量最简单，直接存取即可。
- float 变量可以先转成 int 类型，然后按 int 变量来处理。
- long 变量则需要拆成两个 int 变量（需要注意的点是将两个 int 合并成一个 long 的方式，具体处理方式见下面的源码 getLong）。
- double 变量可以先转成 long 类型，然后按照 long 变量来处理。
- 引用值，也比较简单，直接存取即可。

下面给出 LocalVars 中存取变量或引用的方法

```java
public class LocalVars {
    private Slot[] localVars; //局部变量表,JVM 规定其按照索引访问,所以将其设置为数组

    public LocalVars(int maxLocals) {
        if (maxLocals > 0) {
            localVars = new Slot[maxLocals];
        } else {
            throw new NullPointerException("maxLocals<0");
        }
    }

    //提供了对 int,float,long,double,引用的存取,这里要注意的是 long 和 double 是占用 8 字节的,所以使用了局部变量表中的两个槽位分别存储前四字节和后四字节
    public void setInt(int index, int val) {
        Slot slot = new Slot();
        slot.num = val;
        localVars[index] = slot;
    }

    public int getInt(int index) {
        return localVars[index].num;
    }

    public void setFloat(int index, float val) {
        Slot slot = new Slot();
        slot.num = Float.floatToIntBits(val);
        localVars[index] = slot;
    }

    public float getFloat(int index) {
        return Float.intBitsToFloat(localVars[index].num);
    }

    public void setLong(int index, long val) {
        //先存低 32 位
        Slot slot1 = new Slot();
        slot1.num = (int) (val);
        localVars[index] = slot1;
        //再存高 32 位
        Slot slot2 = new Slot();
        slot2.num = (int) (val >> 32);
        localVars[index + 1] = slot2;
    }

    public long getLong(int index) {
        int low = localVars[index].num;
        long high = localVars[index + 1].num;
        return ((high & 0x000000ffffffffL) << 32) | (low & 0x00000000ffffffffL);
    }

    public void setDouble(int index, double val) {
        long bits = Double.doubleToLongBits(val);
        setLong(index, bits);
    }

    public double getDouble(int index) {
        long bits = getLong(index);
        return Double.longBitsToDouble(bits);
    }

    public void setRef(int index, Zobject ref) {
        Slot slot = new Slot();
        slot.ref = ref;
        localVars[index] = slot;
    }

    public Zobject getRef(int index) {
        return localVars[index].ref;
    }

}
```

注意到，我们这里并没有对 boolean、byte、short 和 char 类型定义存取方法，因为这些类型的值都是转换成 int 值类来处理的。



## 操作数栈

操作数栈的实现方式和局部变量表类似。操作数栈也是通过索引来访问的，操作数栈的大小是编译器已经确定的，所以依然可以用[]Slot 实现。这里有一个 size 字段是用于记录栈顶位置的。

```java
public class OperandStack {

    private int size;  //初始值为 0,在运行中,代表当前栈顶的 index,还未使用,可以直接用,用完记得 size++;
    private Slot[] slots;

    public OperandStack(int maxStack) {
        if (maxStack > 0) {
            slots = new Slot[maxStack];
        } else {
            throw new NullPointerException("maxStack<0");
        }
    }

    public void pushInt(int val) {
        Slot slot = new Slot();
        slot.num = val;
        slots[size] = slot;
        size++;
    }

    public int popInt() {
        size--;
        return slots[size].num;
    }
 	...
}
```

和局部变量表类似，需要定义一些方法从操作数栈中弹出，或者往其中推入各种类型的变量。这里只列举出最简单的对于 int 变量进行存取的的部分，对于其它类型的存取和局部变量表是一样的，就不在浪费篇幅了。这里要注意的一点是：每次存取数据，都要对 size 进行相应的加减操作。



## 局部变量表和操作数栈实例分析

以下面计算圆形周长的方法为例,观察局部变量和操作数栈的变化

```java
public static float circumference(float r) {
    float pi = 3.14f;
    float area = 2 * pi * r;
    return area;
}
```

上述代码被 javac 编译成如下字节码:

```
00 ldc #4
02 fstore_1
03 fconst_2
04 fload_1
05 fmul
06 fload_0
07 fmul
08 fstore_2
09 fload_2
10 return
```

circumference（）方法的局部变量表大小是 3，操作数栈深度是 2;(执行方法所需的局部变量表大小和操作数栈深度是由编译器预先计算好的)

假设调用该方法时传入的参数是 1.6f,那么执行该方法时，操作数栈和局部变量表的变化如下：

![circumference 方法执行过程](http://7xvxof.com1.z0.glb.clouddn.com/jvmgo_40.png)

传递给它的参数是 1.6f，方法开始执行前，1.6 被保存在本地变量表的#0 位置，接下来开始执行字节码：

首先是 ldc，它把 3.14f 推入栈顶

接着是 fstore_1 指令，它把栈顶的 3.14f 弹出，放到#1 号局部变量中

fconst_2 指令把 2.0f 推到栈顶

fload_1 指令把#1 号局部变量推入栈顶

fmul 指令执行浮点数乘法。它把栈顶的两个浮点数弹出，相乘，然后把结果推入栈顶

fload_0 指令把#0 号局部变量推入栈顶

fmul 继续乘法计算

fstore_2 指令把操作数栈顶的 float 值弹出，放入#2 号局部变量表

fload_2 指令把#2 号局部变量推入操作数栈顶

最后 freturn 指令把操作数栈顶的 float 变量弹出，返回给方法调用者。