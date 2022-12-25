---
title: 手写JVM系列(10)-解释器
date: 2017-05-15 11:44:03
tags: JVM
---



> 上一节介绍了关于 JVM 指令集的相关知识，那么接下来我们就根据上一节介绍的知识，编写一个简易的解释器，读取字节码中的操作码和操作数，实现 JVM 规范中描述的指令集对应功能，从而完成简单方法的调用过程，并在接下来的章节中不断完善该解释器，使其支持更多指令。本节所实现的代码均在[项目](https://github.com/zachaxy/JVM)的 instructions 包下。



# 操作码的实现

在 Code 属性中的字节数组就是字节码。操作码规定长度为一个字节，所以这里先读取一个字节，查看其对应的是哪种操作码，然后根据该操作码的实际含义，再决定是不是后面还有操作数需要读取。这一节实现了约 150 条指令，这些指令根据其分类，在 instructions 包下的子包中，具体实现见[项目源码](https://github.com/zachaxy/JVM)。




# 指令抽象接口
有了前面的基础知识，接下来我们就要从字节码中读取指令，并执行指令，完成解释器的功能了。首先定义指令接口，然后定义一个结构体用来辅助指令解码。

```java
public interface Instruction {
    //从字节码中提取操作数
    void fetchOperands(BytecodeReader reader);

    //执行指令逻辑
    void execute(Zframe frame);
}
```

这是所有指令都要完成的两个动作，提取操作数（如果有的话），执行指令。



# 指令抽象类

有很多指令的操作数都是类似的。为了避免重复代码，按照操作码类型定义一些抽象类，这些抽象类实现了 Instruction 接口，并实现 FetchOperands（）方法。再由具体的类去继承这些抽象类，然后
专注实现 Execute（）方法即可。定义的抽象类有以下几种。

##  NoOperandsInstruction

NoOperandsInstruction 表示没有操作数的指令，所以没有定义任何字段。FetchOperands（）方法自然也是空空如也，什么也不用读。

```java
public abstract class NoOperandsInstruction implements Instruction{

    @Override
    public void fetchOperands(BytecodeReader reader) {

    }
}
```


## BranchInstruction

BranchInstruction 表示跳转指令，Offset 字段存放跳转偏移量。FetchOperands（）方法从字节码中读取一个 uint16 整数，转成 int 后赋给 Offset 字段。

```java
public abstract class BranchInstruction implements Instruction {
    public int offset;

    @Override
    public void fetchOperands(BytecodeReader reader) {
        offset = reader.readInt16();
    }
}
```

## Index8Instruction

存储和加载类指令需要根据索引存取局部变量表，索引由单字节操作数给出。把这类指令抽象成 Index8Instruction 结构体，用 Index 字段表示局部变量表索引。FetchOperands（）方法从字节码中读取一个 int8 整数，转成 uint 后赋给 Index 字段。

```java
public abstract class Index8Instruction implements Instruction {

    public Index8Instruction(){}

    public int index;

    @Override
    public void fetchOperands(BytecodeReader reader) {
        index = reader.readUint8();
    }
}
```

## Index16Instruction

有一些指令需要访问运行时常量池，常量池索引由两字节操作数给出。把这类指令抽象成 Index16Instruction 结构体，用 Index 字段表示常量池索引。FetchOperands（）方法从字节码中读取一个 uint16 整数，转成 uint 后赋给 Index 字段。

```java
public abstract class Index16Instruction implements Instruction {
    int index;

    @Override
    public void fetchOperands(BytecodeReader reader) {
        index = reader.readUint16();
    }
}
```



# 读取字节码辅助类

code 字段存放字节码，pc 字段记录读取到了哪个字节。所有的指令都要从字节码中读取数据，并根据不同的指令类型读取 int、long、float、double 等类型的数据，所以定义一个读取字节码的辅助类 BytecodeReader，让其实现一系列的 readXXX（）方法。至于具体调用哪个 read 方法，则根据操作码中对应的操作类型来调用。

```java
public class BytecodeReader {
    byte[] code;  //Java 语言中 byte 的范围四 -128~127,和 Go 语言中的 byte:0~255 不同,所以在取数据的时候需要注意;
    int pc;

    public void reset(byte[] code, int pc) {
        this.code = code;
        this.pc = pc;
    }

    public int getPc() {
        return pc;
    }


    public byte readInt8() {
        byte res = code[pc];
        pc++;
        return res;
    }

    public int readUint8() {
        int res = code[pc];
        res = (res + 256) % 256;
        pc++;
        return res;
    }


    public int readInt16() {
        return (short) readUint16();
    }

    public  int readUint16() {
        int a1 = readUint8();
        int a2 = readUint8();
        return (a1 << 8 | a2);
    }

    public int readInt32() {
        byte[] data = new byte[4];
        data[0] = readInt8();
        data[1] = readInt8();
        data[2] = readInt8();
        data[3] = readInt8();

        return ByteUtils.byteToInt32(data);
    }

    public int[] readInt32s(int n) {
        int[] data = new int[n];
        for (int i = 0; i < n; i++) {
            data[i] = readInt32();
        }
        return data;
    }


    //4k 对齐,没有对齐的会有填充数据,这些数据要忽略掉;
    public void skipPadding() {
        while (pc % 4 != 0) {
            readInt8();
        }
    }
}
```

# 指令创建工厂

有了前面的各个指令的具体实现，接下来统一对外提供一个创建指令的接口，根据不同的操作码创建具体的指令，整体结构就是一个 switch-case 语句，其枚举了这一节已经实现的 150 多种指令，代码太长就不列举了，具体实现在 instructions 包下的 InstructionFactory 类中。



# 解释器

Java 虚拟机解释器的大致逻辑，其伪码如下所示：

```java
do {
    atomically calculate pc and fetch opcode at pc;
    if (operands) fetch operands;
    execute the action for the opcode;
} while (there is more to do);
```

解释一下上面的过程：

1. 计算 pc 值（默认 pc++，取下一条指令，否则就是跳转指令，对应 pc + offset ）；
2. 根据 pc 寄存器的指示位置，从字节码中读取出操作码；
3. 如果该操作码存在操作数，那么继续从字节码中读取操作数；
4. 执行操作码所定义的操作；
5. 如果字节码还未读取完，继续步骤 1；



我们根据前面已经实现的编码，创建 Interpreter 类，实现指令的执行过程，代码如下：

```java
private static void loop(Zthread thread, byte[] byteCode) {
    Zframe frame = thread.popFrame();   //得到栈顶的帧。
    BytecodeReader reader = new BytecodeReader();

    //这里循环的条件是 true,因为在解析指令的时候会遇到 return,而现在还没有实现 return,所以遇到 return 直接抛出异常,那么循环也就终止了
    while (true) {
        int pc = frame.getNextPC(); //这第一次 frame 才刚初始化，获取的 pc 应该是默认值 0 吧。
        thread.setPc(pc);
        reader.reset(byteCode, pc); //reset 方法，其实是在不断的设置 pc 的位置。
        int opCode = reader.readUint8();
        //解析指令,创建指令,然后根据不同的指令执行不同的操作
        Instruction instruction = InstructionFactory.createInstruction(opCode);
        instruction.fetchOperands(reader);
        frame.setNextPC(reader.getPc());

        System.out.printf("pc: %2d, inst: %s \n", pc, instruction.getClass().getSimpleName());
        instruction.execute(frame);
    }
}
```

其中 Interpreter#loop（）方法中，while 循环部分的代码就是 Java 虚拟机规范中 JVM 解释器的具体实现。