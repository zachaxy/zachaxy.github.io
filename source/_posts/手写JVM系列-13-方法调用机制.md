---
title: 手写JVM系列(13)-方法调用机制
date: 2018-01-04 20:46:26
tags: JVM
---

> 我们知道一个程序的运行的逻辑就是在一个代码块中不断的调用方法，将这些方法的调用堆砌好之后就形成的程序的逻辑。有了前面介绍的知识基础，接下来就来介绍下方法的执行过程。本节的代码主要是在[instructions-references包下的 invokeXXX 指令](https://github.com/zachaxy/JVM/tree/master/Java/src/instructions/references)。同时也可以配合[测试用例TestInvokeMethod08](https://github.com/zachaxy/JVM/blob/master/Java/src/test/TestInvokeMethod08.java)加深了解。


# 方法的调用机制

注:方法的调用不等同与方法的执行,方法的调用阶段只是用来确定被调用的是哪一个方法，拿到具体的方法之后，就可以拿到其字节码指令，从而执行程序了。

JVM 中方法调用的指令有以下五条：
- invokestatic:调用静态方法
- invokespecial:调用实例构造器`<init>`方法,私有方法和父类方法
- invokevirtual:调用所有的虚方法
- invokeinterfae:调用接口方法(在运行时确定实现了此接口的一个具体对象)
- invokenative:本地方法的调用

前两个在解析阶段就可以确定唯一的调用版本，而后两个需要在调用时寻找真正的 method。

<!--more-->

# 方法调用执行流程概述
本地方法的执行不在本节讨论之内！
invokeXXX 指令的执行：
(1)首先该系列的指针的操作数是指向运行时常量池的一个索引，由该索引从运行时常量池获取一个方法引用，再将该方法引用转为直接引用
```java
RuntimeConstantPool runtimeConstantPool = frame.getMethod().getClazz().getRuntimeConstantPool();
//通过index,拿到方法符号引用
MethodRef methodRef = (MethodRef) runtimeConstantPool.getRuntimeConstant(index).getValue();
//将 methodRef 转换为 Method
Zmethod resolvedMethod = methodRef.resolvedMethod();
```

(2)获取到resolvedMethod方法之后，需要进行一定的权限验证，例如：invokestatic 就要验证 resolvedMethod 是否是 static 的；还有 resolvedMethod 方法是否可以在外部访问（private，protect）的情况等，如果权限验证不通过，则抛出 IllegalAccessError 异常。

(3)为新方法创建栈帧 newFrame，每个方法的执行都是在自己的 Frame 中，只是创建栈帧，并不压栈，当前栈帧的栈顶依然是调用该方法的栈帧-invokerFrame。

(4)为即将要调用的方法传递参数，这也是方法执行中很重要的一部分，待执行的方法所需的参数由 invokerFrame 的操作数栈提供，此时需要将 invokerFrame 的操作数栈的弹出对应的操作数，放到 newFrame 的本地变量表中。
```java
Zthread thread = invokerFrame.getThread();
Zframe newFrame = thread.createFrame(method);
thread.pushFrame(newFrame);

int argSlotCount = method.getArgSlotCount();
if (argSlotCount > 0) {
    for (int i = argSlotCount - 1; i >= 0; i--) {
        Slot slot = invokerFrame.getOperandStack().popSlot();
        newFrame.getLocalVars().setSlot(i, slot);
    }
}
```
这里需要思考的一个问题是：如果确定方法所需的参数的数量？答案就是根据方法的描述符来确定，同时如果方法是非静态方法，那么其第一个参数就是调用该方法的引用自身，虽然我们并没有在代码中显式的写，但是 Java 编译器在编译成字节码时，帮我们做了。

(5)将 newFrame 压栈，由解释器具体执行具体的方法。


# 多态调用的本质。
通过 invokevirtual 指令调用来进行说明。

```java
class Parent{
	void f1(){}
}

class Child extends Parent{
	void f1(){}
}

public static void main(String[] args){
    Parent p = new Child();
    p.f1();
}
```

上面的 main 方法对应的字节码如下：
```
 0 new #3 <Child>
 3 dup
 4 invokespecial #4 <Child.<init>>
 7 astore_1
 8 aload_1
 9 invokevirtual #5 <Parent.f1>
12 return
```

假设 main 方法所在的栈帧的 frame1，

1. 执行指令 new，该指令后跟一个运行时常量池的索引，通过该索引，获取到一个 ClassRef，其名字是 Child，那么接下来就加载 child 类，并创建该类的一个对象p，并将 p 对象放在 frame1的操作数栈上
2. 执行指令 dup，该指令是将操作数栈顶的slot 复制一份，所以此时 frame1的操作数栈有两个 p 的引用，同时要注意的是这两个 p 指向的是同一块内存的对象。
3. 执行指令 invokespecial，因为上一步 new 只是为 child 对象申请了空间，该对象的各个字段都是类零值，那么下面就要对该对象进行初始化赋值了。特别要注意的是:该 init 方法也是普通的方法，虽然不是我们自己写的，非静态方法在执行时实际的第一个参数是自身，所需的参数从哪里来呢？答案就是 frame1的操作数栈，因为执行init 方法需要参数（该方法没有其它的参数，只有一个隐式的 this），所以从 frame1的操作数弹出一个引用，那么此时 frame1 的操作数栈就只剩下一个 p 的引用了。
4. 接下来创建 frame2 执行 init 方法，由该方法初始化传入的 p 所指向的内存对象，大家不要忘记之前 dup 指令是的 frame1 的操作数栈中有两个 p，这只是引用，指向的同一块内存，所以在 frame2 中将对 p 初始化好之后，frame2被弹栈，此时回到 frame1，其操作数栈的 p 所指向的内存就是被 init 方法初始化好的。
5. 执行指令 astore_1，将操作数栈顶的元素弹出，保存到 frame1的本地变量表，索引为1
6. 执行指令 aload_1，将 frame1的本地变量表索引为1的元素压入操作数栈。此时 frame1的操作数栈为 p ，额。。。
7. 执行指令 invokevirtual，该指令后跟一个运行时常量池索引，通过该索引，获取到一个 methodRef，注意该 methodRef 是 Parent 的，而不是 Child 的，虽然该方法实际上应该就是 child 的，先不管这些。拿到 methodRef 之后，将其转换为 method，然后进行权限验证。通过后寻找真正的 method，从哪里寻找？我们看到这个方法依然是非静态方法，所以依然需要一个 this 的引用，依然需要从 frame1 的操作数栈顶 pop 一个引用，这个引用就是 p，但是我们注意看 p 在第一步时创建的类型是 child，所以接下从 this （也就是 p）中重新寻找 method，得到真正的 method，自然就是 child 的 f1 方法了。这就是多态的本质。
8. 最终执行 return 指令，结束 main 方法的执行。


invoke_virtual 指令的执行过程：
```java
public class INVOKE_VIRTUAL extends Index16Instruction {
    @Override
    public void execute(Zframe frame) {
        //调用该方法所在的类
        Zclass currentClass = frame.getMethod().getClazz();
        RuntimeConstantPool runtimeConstantPool = currentClass.getRuntimeConstantPool();
        //通过index,拿到方法符号引用,虚方法(用到了多态),这个方法引用指向的其实是父类的
        MethodRef methodRef = (MethodRef) runtimeConstantPool.getRuntimeConstant(index).getValue();
        //将方法引用转换为方法
        //这一步拿到解析后的resolvedMethod主要是用来做下面权限的验证;
        //而真正的resolvedMethod是在下面拿到真正的调用者,再次解析到的methodToBeInvoked
        Zmethod resolvedMethod = methodRef.resolvedMethod();

        //从操作数栈中获取调用该非静态方法的引用;参数的传递是从当前frame的操作数栈中根据参数个数,完整的拷贝到调用frame的本地变量表中;
        Zobject ref = frame.getOperandStack().getRefFromTop(resolvedMethod.getArgSlotCount() - 1);
        if (ref == null) {
            throw new NullPointerException("called " + resolvedMethod.getName() + " on a null reference!");
        }

        //相对于invokespecial,本指令还多了这一步,因为ref才是真正的调用者
        //而这次解析到的才是真正的method,这是多态的核心!
        Zmethod methodToBeInvoked = MethodLookup.lookupMethodInClass(ref.getClazz(),
                methodRef.getName(), methodRef.getDescriptor());
        if (methodToBeInvoked == null || methodToBeInvoked.isAbstract()) {
            throw new AbstractMethodError(methodToBeInvoked.getName());
        }



        Zthread thread = invokerFrame.getThread();
        Zframe newFrame = thread.createFrame(method);
        thread.pushFrame(newFrame);

        int argSlotCount = method.getArgSlotCount();
        if (argSlotCount > 0) {
            for (int i = argSlotCount - 1; i >= 0; i--) {
                Slot slot = invokerFrame.getOperandStack().popSlot();
                newFrame.getLocalVars().setSlot(i, slot);
            }
        }
    }
}
```

# 方法的返回
方法的返回是有 xreturn 指令实现的。其中 x 表示返回值的类型，有基本类型和引用类型，当然也有 void 类型。

void 类型的返回是最简单的，直接将当前方法所在的 frame 弹出栈帧即可。
而对于有返回值的return 指令，返回时，返回值在 执行方法的frame 的操作数栈顶，此时需要拿到该返回值 value ，然后弹出当前 frame，回到方法调用的 frame，将 value 放到方法调用 frame 的操作数栈。

以 areturn 指令的执行做一个说明：

```java
Zthread thread = frame.getThread();
Zframe currentFrame = thread.popFrame();
Zframe invokerFrame = thread.getCurrentFrame();
Zobject val = currentFrame.getOperandStack().popRef();
invokerFrame.getOperandStack().pushRef(val);
```