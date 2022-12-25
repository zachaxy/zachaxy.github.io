---
title: 手写JVM系列(17)本地方法的调用过程
date: 2018-01-18 22:26:52
tags: JVM
---

java 中有一些方法前面的关键字是`native`，该关键字修饰的方法表明当前方法不是由 java 实现的方法，而是由本地语言编写的。本地语言就是所在的操作系统所用的语言。因为一些和操作系统底层相关的方法是 java 无法实现的，或者基于性能的考虑，由 java 实现并没有用本地方法实现在执行效率上更高效，所以会用到本地方法。

本地方法的调用是通过 JNI(Java Native Interface)来实现的，JNI 是 JVM 中的一个重要组成部分，但是要让 JVM 支持 JNI 规范还需要做大量的工作，而本系列关于 JVM 的介绍主要还是 JVM 的工作原理，为了不陷入到 JNI 的细节中，同时我们的 JVM 是用 java 来实现的，本身也不具备实现 JNI 规范的能力。因此这里只是通过简单的实例，通过 java 来描述 JNI 的实现过程。本节所实现的代码都在[znative 包下](https://github.com/zachaxy/JVM/tree/master/Java/src/znative)
有关 JNI 的细节，请参考[JNI 系列文章](https://zachaxy.github.io/tags/JNI/)

<!--more-->

# 本地方法的注册
这里提到了注册，一般说起注册，都是在某处进行登记，表明自身的存在。其实在 JVM 中有一个**本地方法注册表**，native 方法的注册也是起一个标记占坑的作用，在本地方法表中注册本地方法，方便后面 JVM 需要调用本地方法时，能够从本地方法注册表中找到对应的本地方法。本节注册本地方法的代码请参考[RegisterCenter.java](https://github.com/zachaxy/JVM/tree/master/Java/src/znative/RegisterCenter.java)


JVM 一般是用 c 语言编写的，在 JVM 的实现中，本地方法其实是在 JVM 中实现的， java 中本地方法都会映射为 JVM 中 c 的一个方法指针。

要实现本地方法的注册，其实就是实现**本地方法注册表**，本地方法注册表本质是上就是一个 map 的数据结构，其 key 为方法所在的类+方法名+方法签名，value 为本地方法的指针。
然而我们用 java 实现的 JVM ，所以只能用接口的方式来模拟 c 中的方法指针。**再次强调用 java 来实现 JVM 是不正规的，但是通过本章的介绍，依然可以对 JVM 本地方法的实现有一个整理的理解。**
用接口的方式模拟 c 中的指针，我们所定义的接口如下：
```java
public interface NativeMethod {
    public void run(Zframe frame);
}
```
定义 NativeMethod 接口，其接口中只有一个 run 方法，用来执行本地方法，同时该方法需要一个 frame 参数，之前介绍 Frame 时，代表一个方法的工作空间，同样本地方法也需要一个工作空间，该 frame 参数就是当前本地方法的工作空间。

下面是本地方法的注册实现：
```java
private static HashMap<String, NativeMethod> nativeMethods = new HashMap<>();

public static void register(String className, String methodName, String methodDescriptor, NativeMethod nativeMethod) {
    String key = className + "~" + methodName + "~" + methodDescriptor;
    nativeMethods.put(key, nativeMethod);
}
```

# 本地方法注册的时机
接下来要考虑的问题在何时进行注册，哪些方法需要注册。其实很多 java 类中都包含 native 方法，而在这些包含 native 方法的类中，一般都有一个 registerNatives 的本地方法，该方法的作用是用来将本类中其它的 native 方法注册到本地方法注册表中。而 registerNatives 方法都是在类的静态代码块中执行的，也就是当前类加载到虚拟机的时候，就将该类的本地方法注册到本地方法注册表中。
例如 `java.lang.Object`中的实现，java 中其它类基本上也是这样的套路。
```java
public class Object {

    private static native void registerNatives();

    static {
        registerNatives();
    }

    ...
}
```

我们的 JVM 只是简易的实现版本，所有的方法都是通过代码手动注册的，所以并没有实现 registerNatives 方法，而是将 Object 中的 native 方法一一手动注册进来。当然可以在遇到 registerNatives 的本地方法时，就将扫描当前类中的 native 方法，依次注册进来。我们的 JVM 就不这样实现了，开篇也也说了我们不想花太多精力在 JNI 的实现细节上，所以我们只是会手动注册有限的几个本地方法，来大致了解一下本地方法调用的整个流程。所以如果遇到了调用 registerNatives 方法的情况，不管其所在的哪个类，就返回一个空的实现。


而我们的 JVM 中所手动注册有限的几个本地方法，都在 RegisterCenter 的 init 方法中手动注册，并在我们的 JVM 开启时，手动调用该方法，即完成了注册。已下就是我的 JVM 中所有实现的本地方法，如果各位感兴趣，可以继续向 init 方法中添加对应的注册。
```java
//对外供 JVM 启动后的唯一接入口，JVM 启动后应该立即调用 RegisterCenter 的 init 方法
public static void init() {
    register("java/lang/Object", "getClass", "()Ljava/lang/Class;", new Nobject.getClass());
    register("java/lang/Class", "getPrimitiveClass", "(Ljava/lang/String;)Ljava/lang/Class;", new Nclass.getPrimitiveClass());
    register("java/lang/Class", "getName0", "()Ljava/lang/String;", new Nclass.getName0());
    register("java/lang/Class", "desiredAssertionStatus0", "(Ljava/lang/Class;)Z", new Nclass.desiredAssertionStatus0());
    register("java/lang/Class", "isArray", "()Z", new Nclass.isArray());
    register("java/lang/Throwable", "fillInStackTrace", "(I)Ljava/lang/Throwable;", new Nthrowable.fillInStackTrace());
}
```


# 为本地方法注入字节码
上面只是完成了本地方法的注册，接下来就要实现本地方法的执行过程了。但是在实现本地方法之前，我们还要对本地方法进行特殊的处理，因为在 JVM 中并没有规定如何实现和调用本地方法。在 JVM 中调用 native 方法和调用其它方法所用的指令是一样的，也就是说调用本地静态方法会产生一个`invoke_static`的指令，调用本地非静态方法，会产生一个`invoke_special`的指令。但是我们对于本地方法，我们要做特殊的处理，因为 **native 方法在 class 文件中没有字节码。** 

以静态普通方法和静态本地方法的指令为例，JVM 都是产生的 invoke_static 的指令，其指令的指令过程：
```java
public class INVOKE_STATIC extends Index16Instruction {
    @Override
    public void execute(Zframe frame) {
        RuntimeConstantPool runtimeConstantPool = frame.getMethod().getClazz().getRuntimeConstantPool();
        //通过 index,拿到方法符号引用
        MethodRef methodRef = (MethodRef) runtimeConstantPool.getRuntimeConstant(index).getValue();
        Zmethod resolvedMethod = methodRef.resolvedMethod();
        if (!resolvedMethod.isStatic()) {
            throw new IncompatibleClassChangeError();
        }

        Zclass clazz = resolvedMethod.getClazz();
        //判断其 Class 是否已经加载过,如果还未加载,那么调用其类的<clinit>方法压栈
        if (!clazz.isInitStarted()) {
            //当前指令已经是在执行 new 了,但是类还没有加载,所以当前帧先回退,让类初始化的帧入栈,等类初始化完成后,重新执行 new;
            frame.revertNextPC();
            ClassInitLogic.initClass(frame.getThread(), clazz);
            return;
        }

        MethodInvokeLogic.invokeMethod(frame, resolvedMethod);
    }
}



public class MethodInvokeLogic {
    public static void invokeMethod(Zframe invokerFrame, Zmethod method) {
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
在 invoke_static 指令中，解析方法符号引用为直接引用，然后就会调用`MethodInvokeLogic.invokeMethod(frame, resolvedMethod)`方法，该方法内部会创建一个新的 frame 来供被调用的方法来执行，此时 JVM 的栈帧的顶部就是该 frame，解释器此时会读取 JVM 栈帧顶部的 frame 作为当前 frame，并执行该 frame 所绑定的 method 中的字节码。因为本地方法没有字节码，但我们又要让本地方法也适应 JVM 解释器，因此我们的做法是为本地方法注入字节码，本地方法的字节码只需要两个字节，第一个字节用来表示接下来要执行的方法是本地方法，(JVM 规范中预留了两条指令，操作码为 0xFE 和 0xFF，这里我们使用 0xFE 来表示本地方法的调用)。第二个字节用来表示本地方法的 return 指令，其 return 指令和非 native 方法的 return 指令完全一样。


接下来先完成本地方法字节码的构造，在创建 Zmethod 实例时，如果该方法是 native 的，我们将进行如下构造：
```java
private Zmethod(Zclass clazz, MemberInfo classFileMethod) {
    super(clazz, classFileMethod);
    copyAttributes(classFileMethod);
    parsedDescriptor = new MethodDescriptor(this.descriptor);
    argSlotCount = calcArgSlotCount(parsedDescriptor.getParameterTypes());
    if (isNative()) {
        injectCodeAttribute(parsedDescriptor.getReturnType());
    }
}

// JVM 并没有规定如何实现和调用本地方法，这里我们依然使用 JVM 栈 来执行本地方法
// 但是本地方法中并不包含字节码，那么本地方法的调用，这里我们利用接口来实现调用对应的方法；
// 同时 JVM 中预留了两条指令，操作码分别是 0xff 和 0xfe，下面使用 0xfe 来当前方法为表示本地方法
// 第二个字节为本地方法的返回指令，该指令和普通方法的返回指令是一样的。
private void injectCodeAttribute(String returnType) {
    //本地方法的操作数栈暂时为 4;至少能容纳返回值
    this.maxStack = 4;
    //本地方法的局部变量表只用来存放参数,因此直接这样赋值
    this.maxLocals = this.argSlotCount;
    //接下来为本地方法构造字节码:起始第一个字节都是 0xfe,用来表用这是本地方法;
    //第二个字节码则根据不同的返回值类型选择相应的 xreturn 的指令即可
    //不必担心下面的 byte 的强转，因为在读取字节码时，使用的是 readUint8()方法
    switch (returnType.charAt(0)) {
        case 'V':
            this.code = new byte[]{(byte) 0xfe, (byte) 0xb1}; // return
            break;
        case 'L':
        case '[':
            this.code = new byte[]{(byte) 0xfe, (byte) 0xb0}; // areturn
            break;
        case 'D':
            this.code = new byte[]{(byte) 0xfe, (byte) 0xaf}; // dreturn
            break;
        case 'F':
            this.code = new byte[]{(byte) 0xfe, (byte) 0xae}; // freturn
            break;
        case 'J':
            this.code = new byte[]{(byte) 0xfe, (byte) 0xad}; // lreturn
            break;
        default:
            this.code = new byte[]{(byte) 0xfe, (byte) 0xac};// ireturn
    }
}
```

注意在上面的字节注入中，还未本地方法添加了本地变量表和操作数栈。本地变量表的大小为本地方法的参数个数。操作数栈的大小这里设置为 4，其实这个值至少是 2，因为我们只是用操作数栈来盛放本地方法的返回值。



# 本地方法的调用

在完成了本地方法的字节注入之后，终于可以实现本地方法的调用了。因为我们为本地方法注入了两个字节码，所以在 JVM 的解释器真正执行本地方法时，会本地方法中的这两个字节码，首先是 0xFE，这是我们自己构造的，解释器在遇到这个字节码时，其实是不认识的，因为 JVM 规范中没有用到该字节码，于是在这里我们手动创建一个 invoke_native 指令，该指令就对应 0xFE。不要忘了在我们的 InstructionFactory.java 的 createInstruction 方法中，添加对该字节码的解析。注意：JVM 中是没有 invoke_native 指令的，这是我们自己创建的指令。其具体实现如下：
```java
public class INVOKE_NATIVE extends NoOperandsInstruction {
    @Override
    public void execute(Zframe frame) {
        Zmethod method = frame.getMethod();
        String clazzName = method.getClazz().thisClassName;
        String methodName = method.getName();
        String descriptor = method.getDescriptor();

        NativeMethod nativeMethod = RegisterCenter.findNativeMethod(clazzName, methodName, descriptor);
        if (nativeMethod == null) {
            String methodInfo = clazzName + "." + methodName + descriptor;
            throw new UnsatisfiedLinkError(methodInfo);
        }

        nativeMethod.run(frame);
    }
}
```
上面也说了本地方法中没有字节码，我们认为注入了标示本地方法开始的字节码 0xFE，已经本地方法返回的字节码。但是中间具体的指令流程我们是无法用字节码构造了，因为这样太麻烦了。所以在 java 层面是不知道本地方法内部如何执行，因为本地方法会映射到 JVM 中的一个 c 的指针，也就是说每一个本地方法对应的具体实现都已经在 JVM 中实现好了，由该指针指向的代码来执行具体的逻辑(我们的 JVM 中就是通过接口来模拟方法指针了)。这就需要在 JVM 找到对应的本地方法的具体实现，并执行本地方法。

本地方法的查找，也很简单，只要从本地方法注册表中根据方法名获取到具体实现即可，我们这里是获取到 NativeMethod 接口的一个实现类，然后调用其 run 方法。这里要注意的是如果方法名为 registerNatives,则返回一个空实现即可，原因在文章开始已经解释过了。对于其它情况，如果在本地方法表中没有提前注册，那么将抛出 UnsatisfiedLinkError 的错误。

```java
public static NativeMethod findNativeMethod(String className, String methodName, String methodDescriptor) {
    String key = className + "~" + methodName + "~" + methodDescriptor;
    if (nativeMethods.containsKey(key)) {
        return nativeMethods.get(key);
    }

    if ("()V".equals(methodDescriptor)) {
        if ("registerNatives".equals(methodName)) {
            //返回一个空的方法执行体 emptyNativeMethod
            return new NativeMethod() {
                @Override
                public void run(Zframe frame) {

                }
            };
        }
    }

    return null;
}
```


# 本地方法调用实战
通过上面的讲解，可能还是一头雾水，请不要怀疑自己的能力，是我表达的不太够准确，因此在最后一节，准备了一个本地方法调用的实战，当你看明白了这个实战的流程之后，再看上面的讲解，也许你会豁然开朗。

本实战准备的例子是[TestGetClass11.java](https://github.com/zachaxy/JVM/blob/master/Java/src/test/TestGetClass11.java)
测试所用的方法是：
```java
public static void test() {
    String s1 = int.class.getName();
}
```

这里是调用了基本类型的类，[上一章](https://zachaxy.github.io/2018/01/18/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-16-%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6%E7%AE%80%E4%BB%8B/)反射中已经介绍了其实现原理，我们这里重点看 getName 方法，该方法内步流程是：
```java
public String getName() {
    String name = this.name;
    if (name == null)
        this.name = name = getName0();
    return name;
}

private native String getName0();
```

发现这里调用了 getName0 方法，而 getName0 方法就是本地方法。所以我们要在 JVM 中实现该方法。在 znative 包下，创建 Nclass.java，在 java/lang/Class 中的本地方法都将在 Nclass 中进行实现。不过我们并不打算将 Class 中的本地方法全部实现，这里只用到了 getName0 的方法，所以我们只实现 getName0 的逻辑，其具体实现如下：
```java
public static class getName0 implements NativeMethod {
    @Override
    public void run(Zframe frame) {
        Zobject self = frame.getLocalVars().getRef(0);
        Zclass clazz = (Zclass) self.extra;
        String name = clazz.getJavaName();
        Zobject nameObj = StringPool.jString(clazz.getLoader(), name);
        frame.getOperandStack().pushRef(nameObj);
    }
}
```

在 JVM 本地方法 getName0 实现了 NativeMethod 接口，在 run 方法中，首先获取本地变量表中的第一个变量，也就是调用 getName0 方法的对象 `this`，然后获取到 this 的类，最核心就是 `class.getJavaName()`这一句，获取到当前 class 的 javaName，其实现也非常简单，在 Zclass.java 中
```java
public String getJavaName() {
    return thisClassName.replace("/", ".");
}
```

有了本地方法的具体实现，接下来记得在本地方法注册表中进行注册，其在 RegisterCenter.java 的 init 方法中：
```java
public static void init() {
	...
	register("java/lang/Class", "getName0", "()Ljava/lang/String;", new Nclass.getName0());
	...
}
```

最后不要忘记在 JVM 启动之前，调用`RegisterCenter.init()`，将本地方法注册到 JVM 中。具体实现流程请参考[TestGetClass11.java](https://github.com/zachaxy/JVM/blob/master/Java/src/test/TestGetClass11.java)