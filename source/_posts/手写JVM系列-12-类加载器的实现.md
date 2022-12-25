---
title: 手写JVM系列(12)-类加载器的实现
date: 2018-01-04 15:32:52
tags: JVM
---

> 通过之前的文章，我们了解了 Class 文件存储格式的具体细节，在 Class 文件中描述的各种信息,最终都需要加载到虚拟机中之后才能运行和使用。之前我们已经可以将 Class 文件的字节码读取到内存中了，这其实就是类加载器中 的加载过程，属于类加载器的一部分功能，除此之外，类加载器还有其它功能需要完善，这一节将对类加载的过程做一个详细的介绍。
> 本节的代码集中在 [heap](https://github.com/zachaxy/JVM/tree/master/Java/src/runtimedata/heap) 包下。

# 类加载的生命周期

类从被加载到虚拟机开始，到卸载出内存，其整个生命周期包括：
1. 加载
2. 验证----|
3. 准备----|—>连接
4. 解析----|
5. 初始化
6. 使用
7. 卸载

其中：验证，准备，解析3个阶段统称为连接阶段。C语言需要在编译时进行连接工作，但是Java不同，其“连接”的工作全都推迟到运行期来做，这虽然会增加一些性能上的开销，但是却为java应用程序提供高度的灵活性.

其生命周期的 加载，验证，准备，初始化，卸载这五个阶段顺序是确定的，而类的解析阶段，正常情况下是在初始化之前的，但是遇到多态，则在初始化之后开始。这也充分体现了Java的灵活性。
我们重点要掌握的是加载，验证，准备，解析,初始化这五个阶段。

<!--more-->

虚拟机规范严格规定有且只有以下5种情况必须立即对类进行初始化:（前提是该类还未进行过初始化，以下5个条件任何一个都会触发其初始化动作；如果已经进行过初始化，会从 **方法区** 中直接读取）
1. 遇到new，getstatic，putstatic，invokestatic这四条字节码指令时；其分别对应Java语法中的使用new关键字实例化对象，读写一个类的静态变量时（若是读的static final 的基本类型或者字符串，那么其值是在字段的ConstantValue属性中就初始化好的，并不需要对类进行初始化），调用类的静态方法时；
2. 使用反射对类进行调用时
3. 初始化一个类的时候，发现其父类还未初始化，那么先触发其父类的初始化
4. 启动虚拟机时，需要制定一个包含main方法的类，虚拟机会先初始化该类
5. 如果一个`MethodHandle`实例最后的解析结果是 `REF_getStatic`,`REF_putStatic`,`REF_invokeStatic` 的方法的句柄时，会触发对应的 REF 类加载。


# 加载
加载阶段，虚拟机需要完成2件事：
1. 通过类的全限定名来获取该类的二进制字节流，对应我们之前讲的搜寻类所在的路径，并读取 class 文件，将其映射为 [ClassFile](https://github.com/zachaxy/JVM/blob/master/Java/src/classfile/ClassFile.java) 对象
2. 将这个字节流所代表的静态存储初结构转换为方法区的运行时数据结构，将 ClassFile 转换为 [Zclass](https://github.com/zachaxy/JVM/blob/master/Java/src/runtimedata/heap/Zclass.java)


```java
//读取 class 文件
private byte[] readClass(String name) {
    byte[] data = classPath.readClass(name);
    if (data != null) {
        return data;
    } else {
        return null;
    }
}

// 将上面读取的到字节码转换为最终的 Zclass
private Zclass paraseClass(byte[] data) {
    ClassFile cf = new ClassFile(data);
    return new Zclass(cf);
}
```


# 验证
这一阶段的目的是确保Class文件中的字节流包含的信息符合当前虚拟机的要求。该阶段需要接受四小阶段的验证动作：
1. 文件格式验证：是否按照Class字节码的格式组织的，同时对索引的访问是否越界等
2. 元数据验证：对字节码（Code中的属性）描述的信息进行语义分析，以保证其符合Java语言规范
3. 字节码验证：对数据流和控制流分析，确定程序语义是合法的，而不会在运行时危害虚拟机安全
4. 符号引用验证：在解析阶段发生，主要是验证符号引用转换为直接引用的时候。若无法通过符号引用验证，那么将抛出IncompatibleClassChangeError的子类。符号引用验证在转换为直接引用的时候，还要判断当前类是都对转换的直接引用有访问权限，如果没有，则抛出`java.lang.IllegalAccessError`异常。

注意：这个阶段并不是必须的，如果是我们自己写的代码，后者大量验证了第三方库的代码是无害的，那么完全可以取消验证阶段，以提高类加载的效率。`可使用 -Xverify:none 参数来关闭验证`

而在本 JVM 中并没有对其进行验证。

```java
//在执行类的任何代码之前要对类进行严格的检验,这里忽略检验过程,,作为空实现;
private void verify(Zclass clazz) {
}
```

# 准备
准备阶段是为类的**静态变量**分配内存，并设置初值的阶段，这里要注意的是，**设置初值指的是类型的零值**。
例如类中有如下语句：`public static int i = 123;`，那么i在初始化阶段过后的值为0，而不是123，赋值为123的阶段是在后面的初始化阶段。
但是如果是同时被final修饰变量，`public static final int i = 123;`那么会为该变量生成ConstantValue属性，并在准备阶段将其值设为123。

```java
//给类变量分配空间并赋予初始值
private void prepare(Zclass clazz) {
    calcInstanceFieldSlotIds(clazz);
    calcStaticFieldSlotIds(clazz);
    allocAndInitStaticVars(clazz);
}

// 计算new一个对象所需的空间,单位是clazz.instanceSlotCount,主要包含了类的非静态成员变量(包含父类的)
// 但是这里并没有真正的申请空间，只是计算大小，同时为每个非静态变量关联 slotId
private void calcInstanceFieldSlotIds(Zclass clazz) {
    int slotId = 0;
    if (clazz.superClass != null) {
        slotId = clazz.superClass.instanceSlotCount;
    }

    for (Zfield field : clazz.fileds) {
        if (!field.isStatic()) {
            field.slotId = slotId;
            slotId++;
            if (field.isLongOrDouble()) {
                slotId++;
            }
        }
    }
    clazz.instanceSlotCount = slotId;
}


//计算类的静态成员变量所需的空间，不包含父类，同样也只是计算和分配 slotId，不申请空间
private void calcStaticFieldSlotIds(Zclass clazz) {
    int slotId = 0;
    for (Zfield field : clazz.fileds) {
        if (field.isStatic()) {
            field.slotId = slotId;
            slotId++;
            if (field.isLongOrDouble()) {
                slotId++;
            }
        }
    }
    clazz.staticSlotCount = slotId;
}

// 为静态变量申请空间,注意:这个申请空间的过程,就是将所有的静态变量赋值为0或者null;
// 如果是 static final 的基本类型或者 String，其值会保存在ConstantValueAttribute属性中
// 而ConstantValueAttribute属性中保存的值又是在常量池中！
private void allocAndInitStaticVars(Zclass clazz) {
    clazz.staticVars = new Slots(clazz.staticSlotCount);
    for (Zfield field : clazz.fileds) {
        if (field.isStatic() && field.isFinal()) {
            initStaticFinalVar(clazz, field);
        }
    }
}


// 为static final 修饰的成员赋值,这种类型的成员是ConstantXXXInfo类型的,该info中包含真正的值在运行时常量池中;
private void initStaticFinalVar(Zclass clazz, Zfield zfield) {
    Slots staticVars = clazz.staticVars;
    RuntimeConstantPool runtimeConstantPool = clazz.getRuntimeConstantPool();
    int index = zfield.constValueIndex;
    int slotId = zfield.slotId;

    if (index > 0) {
        switch (zfield.getDescriptor()) {
            case "Z":
            case "B":
            case "C":
            case "S":
            case "I":
                int intValue = (int) runtimeConstantPool.getRuntimeConstant(index).getValue();
                staticVars.setInt(slotId, intValue);
                break;
            case "J":
                long longValue = (long) runtimeConstantPool.getRuntimeConstant(index).getValue();
                staticVars.setLong(slotId, longValue);
                break;
            case "F":
                float floatValue = (float) runtimeConstantPool.getRuntimeConstant(index).getValue();
                staticVars.setFloat(slotId, floatValue);
                break;
            case "D":
                double doubleValue = (double) runtimeConstantPool.getRuntimeConstant(index).getValue();
                staticVars.setDouble(slotId, doubleValue);
                break;
            case "Ljava/lang/String;":
                String stringValue = (String) runtimeConstantPool.getRuntimeConstant(index).getValue();
                Zobject jStr = StringPool.jString(clazz.getLoader(), stringValue);
            default:
                break;
        }
    }
}
```

# 解析
解析阶段在[上一节](https://zachaxy.github.io/2018/01/03/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-11-%E7%BA%BF%E7%A8%8B%E5%85%B1%E4%BA%AB%E7%9A%84%E8%BF%90%E8%A1%8C%E6%97%B6%E6%95%B0%E6%8D%AE/#%E6%80%BB%E7%BB%93)分析过概念，而接下会通过代码进行解释。该阶段是将class文件中常量池的符号引用替换为直接引用的过程，符号引用指的是在class文件中的`Constant_Class_info`,`Constant_Fieldref_info`,`Constant_Methodref_info`,`Constant_InterfaceMethodref_info`这四个常量。以及`Constant_MethodType_info`，`Constant_MethodHandle_info`，`Constant_Invokedynamic_info`。而这三种常量类型与JDK1.7新增的动态语言支持相关，这里暂时不涉及。


符号引用vs直接引用
在class文件中，是一堆数据，这四个常量类型是以字符串来描述的，符号引用与虚拟机中的内存布局无关，因此成为符号引用。而直接引用则是直接指向虚拟机内存中目标的指针。

上述四种常量类型可分为三类：
- 类或接口
- 字段
- 方法
  接下来对这三种类型的解析过程做一个介绍

## 类或接口类型的解析
当前类d，以及其class文件中的常量池，解析其中的`Constant_Class_info`，根据该常量，可以获取其对应的全限定名，然后将全限定名传给d的类加载器，去接在目标类c，中途出现任何异常，都将终止整个解析过程。将c加载出来后，还要判断当前类d对c是否具有访问权限(该过程属于符号引用验证阶段)，如果没有访问权限，则抛出`java.lang.IllegalAccessError`异常。

## 字段的解析
当前类d，以及其class文件中的常量池，解析其中的`Constant_Fieldref_info`，根据该常量，首先获取其classIndex索引，获取到常量池中`Constant_Class_info`，对其进行解析，如果目标类c解析成功，接下来进行后续的字段搜索。
1. 如果c中包含了该目标字段(简单名称和字段描述符都能匹配)，则返回该字段的直接引用。
2. 否则，如果c实现了接口，会按照继承关系，从下向上递归搜索接口和父接口，如果能找到，则返回该字段的引用
3. 否则，按照类继承关系，从下向上递归搜索父类，直到顶级父类Object，如果能找到，则返回该字段的引用
4. 否则，查找失败，抛出`java.lang.NoSuchFieldError`异常。

如果查找过程中成功返回了引用，则将对目标字段进行权限验证，如果没有访问权限，则抛出`java.lang.IllegalAccessError`异常。


## 类方法的解析
当前类d，以及其class文件中的常量池，解析其中的`Constant_Methodref_info`，根据该常量，首先获取其classIndex索引，获取到常量池中`Constant_Class_info`，对其进行解析，如果目标类c解析成功，接下来进行后续的方法搜索。
1. 首先明确这是类方法，而不是接口方法。如果在类方法中发现classIndex索引的c是个接口，那么直接抛出`java.lang.IncompatibleClassChangeError`异常
2. 如果通过第1步，在c中查找是否有简单名称和描述符都与目标匹配的方法，如果有则返回该方法的直接引用
3. 否则，按照类继承关系，从下向上递归搜索父类，直到顶级父类Object，如果能找到，则返回该方法的引用
4. 否则，在c实现的接口和其父接口中递归查找简单名称和描述符都与目标匹配的方法，如果有，说明c是一个抽象类，抛出`java.lang.AbstractMethodError`异常
5. 否则，查找失败，抛出`java.lang.NoSuchMethodError`

如果查找过程成功返回了引用，则将对目标方法进行权限验证，如果没有访问权限，则抛出`java.lang.IllegalAccessError`异常。

## 接口方法的解析
当前类d，以及其class文件中的常量池，解析其中的`Constant_InterfaceMethodref_info`，根据该常量，首先获取其classIndex索引，获取到常量池中`Constant_Class_info`，对其进行解析，如果目标接口 c解析成功，接下来进行后续的方法搜索。
1. 首先明确这是接口方法，而不是类方法。如果在接口方法中发现classIndex索引的c是个类，那么直接抛出`java.lang.IncompatibleClassChangeError`异常
2. 如果通过第1步，在c中查找是否有简单名称和描述符都与目标匹配的方法，如果有则返回该方法的直接引用
3. 否则，按照接口继承关系，从下向上递归搜索父接口，直到顶级父类Object(接口的顶级父类也是Object)，如果能找到，则返回该方法的引用
4. 否则，查找失败，抛出`java.lang.NoSuchMethodError`

接口中的方法默认都是public的，所以不存在访问权限问题。


## 以类引用的解析为例
```
public class SymRef {
    RuntimeConstantPool runtimeConstantPool;   //存放符号引用所在的运行时常量池指针,可以通过符号引用访问到运行时常量池，进一步又可以访问到类数据
    String className;   //存放类的完全限定名
    Zclass clazz;       //上述运行时常量池的宿主类中的符号引用的真正类,在外面访问时，根据 clazz 是否为 null 来决定是否执行 loadClass

    public SymRef(RuntimeConstantPool runtimeConstantPool) {
        this.runtimeConstantPool = runtimeConstantPool;
    }

    //类引用转直接引用
    public Zclass resolvedClass() {
        if (clazz == null) {
            resolvedClassRef();
        }
        return clazz;
    }

    // 当前类(runtimeConstantPool的宿主类)d中,如果引用了类c,那么就将c加载进来
    private void resolvedClassRef() {
        Zclass d = runtimeConstantPool.clazz;
        Zclass c = d.loader.loadClass(className);
        //在这里判断下 d 能否访问 c
        if (!c.isAccessibleTo(d)) {
            throw new IllegalAccessError(d.thisClassName + " can't access " + c.thisClassName);
        }
        clazz = c;
    }
}
```
上面的 SymRef 代表类引用，其核心方法就是resolvedClassRef方法。该方法是在什么情况下使用呢？

# 初始化
初始化阶段才是真正执行类中定义的Java程序代码(字节码)。
在前面的准备阶段，已经对类的静态变量进行过一次赋值了(0或者final定义的值)。而在初始化阶段，则根据程序员通过程序制定的代码去初始化类静态变量。其实，初始化阶段就是执行类构造器 `<clinit>`方法的过程。

`<clinit>`方法由编译器自动收集类中所有静态变量的赋值语句和静态代码块的语句，按照其在源文件中出现的顺序融到`<clinit>`中。同时，虚拟机会保证在执行子类的`<clinit>`方法之前，父类的`<clinit>`方法已经执行完毕。所以虚拟机中第一个被执行的`<clinit>`方法一定是Object类的。

注意：只有当类中存在静态变量赋值语句或者静态代码块时，才会产生`<clinit>`方法，如果没有这些，那么虚拟机也就没有必要去创建`<clinit>`方法。


接口中虽然不能包含静态代码块，但是可以包含变量的赋值，而且接口中的变量默认是`public static final` 的。因此接口中也可以有`<clinit>`方法，但是执行接口的`<clinit>`方法并不要求父接口的`<clinit>`先执行。

初始化阶段执行的`<clinit>`方法我们当做一个普通的方法来执行就可以了，如果有的话，其在 class 文件中会产生对应的字节码。但是唯一要注意的是，`<clinit>`方法同一个类只能执行一次。因此在 Zclass 类中，添加了一个布尔类型的 initStarted 字段判断类是否已经初始化，执行了类的`<clinit>`方法。

回想文章一开始介绍的类加载的时机，是在遇到new，getstatic，putstatic，invokestatic这四条字节码指令时，那么我们就在这四个指令的执行过程中去判断一个 class 是否已经被初始化，如果没有执行过，那么先执行其类初始化方法。
```java
//判断其Class是否已经加载过,如果还未加载,那么调用其类的<clinit>方法压栈
if (!clazz.isInitStarted()) {
    //当前指令已经是在执行new了,但是类还没有加载,所以当前帧先回退,让类初始化的帧入栈,等类初始化完成后,重新执行new;
    frame.revertNextPC();
    ClassInitLogic.initClass(frame.getThread(), clazz);
    return;
}
```

# 类加载器的实现
具体源码请参考[ZclassLoader](https://github.com/zachaxy/JVM/blob/master/Java/src/runtimedata/heap/ZclassLoader.java)，这里简述下其大体流程：

```java
public class ZclassLoader {
    ClassPath classPath;
    //作为缓存，之前加载过这个类，那么就将其class引用保存到map中，后面再用到这个类的时候，直接用map中取；
    HashMap<String, Zclass> map;  

    public ZclassLoader(ClassPath classPath) {
        this.classPath = classPath;
        this.map = new HashMap<String, Zclass>();

        loadBasicClasses();
        loadPrimitiveClasses();
    }
    
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
        ......
        return clazz;
    }   
    
    private Zclass loadNonArrayClass(String name) {
        byte[] data = readClass(name);
        Zclass clazz = defineClass(data);
        link(clazz);
        return clazz;
    }  
    

    private byte[] readClass(String name) {
        byte[] data = classPath.readClass(name);
        if (data != null) {
            return data;
        } else {
            throw new ClassCastException("class name: " + name);
        }
    }
    
    
    private Zclass defineClass(byte[] data) {
        Zclass clazz = parseClass(data);
        clazz.loader = this;
        resolveSuperClass(clazz);
        resolveInterfaces(clazz);
        map.put(clazz.thisClassName, clazz);
        return clazz;
    }
    
   private void link(Zclass clazz) {
        verify(clazz);
        prepare(clazz);
    }

    //在执行类的任何代码之前要对类进行严格的检验,这里忽略检验过程,作为空实现;
    private void verify(Zclass clazz) {
    }

    //给类变量分配空间并赋予初始值
    private void prepare(Zclass clazz) {
        calcInstanceFieldSlotIds(clazz);
        calcStaticFieldSlotIds(clazz);
        allocAndInitStaticVars(clazz);
    }
}
```