---
title: 手写JVM系列(11)-线程共享的运行时数据
date: 2018-01-03 16:42:03
tags: JVM
---

> 前面已经初步实现了线程私有的运行时数据区，主要包括操作数栈和局部变量表。接下来将继续丰富运行时数据区的内容——线程共享的运行时数据区，包括方法区和运行时常量池。本节的代码集中在 [heap](https://github.com/zachaxy/JVM/tree/master/Java/src/runtimedata/heap) 包下。同时也会对一些概念做出解释。因为自己之前在这些概念上的错误认识，走了很多弯路。

<!--more-->

# 方法区
之前我们已经可以找到 class 文件，并把其内容加载到内存中，对其解析成一个 ClassFile 的结构，但是 ClassFile 中的内容仍然无法直接在方法区使用，还需要进一步的转换。其实转换的内容不多，我们通过代码来直观的对已一下 ClassFile 和 Zclass 中的区别。(为避免和 JDK 中已有的类名冲突，这里将我们自定义的 `class`，`method`，`field` 等前面都加一个 `z` 以作区分。)

```java
public class Zclass {
    private int accessFlags;        // 表示当前类的访问标志
    public String thisClassName;   //当前类名字(完全限定名)
    public String superClassName;  //父类名字(完全限定名)
    public String[] interfaceNames;//接口名字(完全限定名,不可以为null,若为实现接口,数组大小为0)
    private RuntimeConstantPool runtimeConstantPool;//运行时常量池,注意和class文件中常量池区别;
    Zfield[] fileds;        //字段表,包括静态和非静态，此时并不分配 slotId；下面的staticVars 是其子集
    Zmethod[] methods;      //方法表，包括静态和非静态
    ZclassLoader loader;    //类加载器
    Zclass superClass;      //当前类的父类class,由类加载时,给父类赋值;
    Zclass[] interfaces;    //当前类的接口class,由类加载时,给父类赋值;
    int instanceSlotCount;  //非静态变量占用slot大小,这里只是统计个数(从顶级父类Object开始算起)
    int staticSlotCount;    // 静态变量所占空间大小
    Slots staticVars;      // 存放静态变量
    
}
```

```java
public class ClassFile {
    private int minorVersion;
    private int majorVersion;
    public ConstantPool constantPool;
    private int accessFlags;
    private int thisClass;
    private int superClass;         //同 thisClass 的索引值。
    private int[] interfaces;     //存放所实现的接口在常量池中的索引。同 thisClass 的索引值。
    private MemberInfo[] fields;    //存放类中所有的字段，包括静态的非静态的；不同的属性通过字段的访问修饰符来读取；
    private MemberInfo[] methods;   //存放类中所有的方法，包括静态的非静态的；不同的属性通过方法的访问修饰符来读取；
    private AttributeInfo[] attributes; //属性表，存放类的属性；
}
```

对比由 ClassFile -> Zclass 的转换可以发现：eg：ClassFile 中  superClass 的值是一个 int 的索引值，其指向常量池的一个常量，该常量由通过一个索引指向常量池中的 utf8-constant，从而用来表示当前 class 的父类的名字(字符串)。而在 Zclass 中，可以看到有一个 superClass 的成员变量，其类型为 Zclass 类型，而不再是简单的字符串。再来看在类中定义的成员变量，在 ClassFile 中成员变量是保存在MemberInfo 中，这也是简单的字符描述(变量类型，变量名字),在 Zclass 中保存的是新定义的 Zfield 类型的变量，这些都是需要转换的地方。

所以转换的本质是什么：
** class 文件中全是字符串，而现在加载到内存中了，就不能简单的用一个字符串来描述了，而是一个指向内存的一个实体**

所以针对于 ClassFile 到 Zclass 的转换，具体要转换的有三部分：

- 类信息本身，这里定义为 Zclass
- 字段信息，这里定义为 Zfield，表示在类中定义的变量(静态+非静态)，Zfield 中有一个成员变量 Zclass，表明当前字段属于哪个类
- 方法信息，这里定义为 Zmethod，表示在类中定义的方法(静态+非静态)，Zmethod 中有一个成员变量 Zclass，表明当前方法属于哪个类

而转换为 Zclass 的类，就是放在方法区的，其实叫方法区是有干扰的，让人以为方法区只是存放类中的方法的，并不是这样的。方法区存放的是一个 class 文件的描述，包括该类的权限，该类实现了哪些接口，该类的父类是谁，该类有哪些字段，以及字段的权限，该类有哪些方法，以及方法的权限等。
在本 JVM 中，方法区的实现是使用了一个 HashMap，其中 key 的 class 的全限定名，value 为加载进来的 zclass 对象。

读完上面的描述，你可能还是一头雾水，暂时先放到一边，因为单将方法区无法将清楚，其还要配合下面的运行时常量，才能查看其全貌。



# 运行时常量池
常量池，这个概念之前就遇到过，那是在解析 class 文件时，class 文件中就有一个常量池的概念，现在又遇到了常量池，不过不是一回事。

这里再次解释一下 class 文件中的常量池的概念。我们将 java 代码通过 javac 编译为 class 文件，接下来再运行 java 程序的时候就完全不需要 java 文件了，只需要 class 文件即可，但是 class 文件是保存在本地磁盘上的文件，里面全是0101的字节码，或者说是字符串，那各个字符串之间如何产生联系呢？那就是通过常量池，并给常量池安排了序号，每个序号都对应一个常量(字符串)，然后 class 文件规定了一个格式，每个 class 文件都要按照一定的次序排放，类名，访问权限，成员变量，成员方法等(具体可回顾[分析class文件](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-4-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6/))。如何描述类名，成员变量，成员方法？就是在 class 文件中对应的位置保存一个整数，用来指向 class 文件中另一块区域——常量池中的一个常量(字符串)，用字符串来描述这是什么。
在强调一遍：class 文件是在本地磁盘上的文件，其内部是死的。

那么接下来将 class 文件读到内存中，其常量池也要进行相应的转换。如果是基本类型的常量那么原封不动，但如果表示的是一个类的引用，在 class 文件中使用的还是字符串，用来说明是哪个类，而在运行期间，就要真正的加载这个类(保存在方法区)，然后指向加载的这个类，而不是之前简单的字符串了。


到这里，应该对两个常量池的作用有一个清楚的了解，自己之前就是把两个常量池当成一个，直接拿来用了！所以在描述常量池时，我会用 `class 文件中的常量池` 和 `运行时常量池` 来做区分。


所以，相对于 class 文件中的常量池，运行时常量池是在内存中的，是活的。由 class 文件中的常量池向运行时常量池的转换如下：
```java
//主要作用是将class文件中的常量池转换为运行时常量池;
public RuntimeConstantPool(Zclass clazz, ConstantPool classFileConstantPool) {
    this.clazz = clazz;
    ConstantInfo[] classFileConstantInfos = classFileConstantPool.getInfos();
    int len = classFileConstantInfos.length;
    this.infos = new RuntimeConstantInfo[len];
    for (int i = 1; i < len; i++) {
        ConstantInfo classFileConstantInfo = classFileConstantInfos[i];
        switch (classFileConstantInfo.getType()) {
            case ConstantInfo.CONSTANT_Integer:
                ConstantIntegerInfo intInfo = (ConstantIntegerInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<Integer>(intInfo.getVal(), ConstantInfo.CONSTANT_Integer);
                break;
            case ConstantInfo.CONSTANT_Float:
                ConstantFloatInfo floatInfo = (ConstantFloatInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<Float>(floatInfo.getVal(), ConstantInfo.CONSTANT_Float);
                break;
            case ConstantInfo.CONSTANT_Long:
                //Long 和 Double 在转换结束之后，都要进行 i++,以适配 class 文件中常量池的索引
                ConstantLongInfo longInfo = (ConstantLongInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<Long>(longInfo.getVal(), ConstantInfo.CONSTANT_Long);
                i++;
                break;
            case ConstantInfo.CONSTANT_Double:
                ConstantDoubleInfo doubleInfo = (ConstantDoubleInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<Double>(doubleInfo.getVal(), ConstantInfo.CONSTANT_Double);
                i++;
                break;
            case ConstantInfo.CONSTANT_String:
                //在对字符串引用进行转换的时候，转为字符串直接引用
                ConstantStringInfo stringInfo = (ConstantStringInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<String>(stringInfo.getString(), ConstantInfo.CONSTANT_String);
                break;
            case ConstantInfo.CONSTANT_Class:
                ConstantClassInfo classInfo = (ConstantClassInfo) classFileConstantInfo;
                //ref 类中真正需要的是 传入上面的 clazz
                this.infos[i] = new RuntimeConstantInfo<ClassRef>(new ClassRef(this, classInfo), ConstantInfo.CONSTANT_Class);
                break;
            case ConstantInfo.CONSTANT_Fieldref:
                ConstantFieldRefInfo fieldRefInfo = (ConstantFieldRefInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<FieldRef>(new FieldRef(this, fieldRefInfo), ConstantInfo.CONSTANT_Fieldref);
                break;
            case ConstantInfo.CONSTANT_Methodref:
                ConstantMethodRefInfo methodRefInfo = (ConstantMethodRefInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<MethodRef>(new MethodRef(this, methodRefInfo), ConstantInfo.CONSTANT_Methodref);
                break;
            case ConstantInfo.CONSTANT_InterfaceMethodref:
                ConstantInterfaceMethodRefInfo interfaceMethodRefInfo = (ConstantInterfaceMethodRefInfo) classFileConstantInfo;
                this.infos[i] = new RuntimeConstantInfo<InterfaceMethodRef>(new InterfaceMethodRef(this, interfaceMethodRefInfo), ConstantInfo.CONSTANT_InterfaceMethodref);
                break;
            default:
                //还有一些jdk1.7才开始支持的动态属性,不在本虚拟机的实现范围内
                break;

        }
    }
}
```

这里主要是对以下四种常量进行转换：
- 类符号引用
- 字段符号引用
- 方法符号引用
- 接口方法符号引用

这里再区分一个概念，拿字段符号引用来说，什么是字段符号引用，和上面讲到的方法区中要转换的字段又有什么关系？
以下面一个简单的例子说明：

```java
class A{
    String str;
    void f(){
        System.out.println("xxx");
    }
}
```
这个例子中 str 就是之前方法区要转换的字段。System.out 是字段符号引用,因为该 out 字段并非类 A 中定义的，而是在 System 类中定义的。

二者相同点：都是字段，那么是字段就要有字段名，类型，所在的类

不同的是：
方法区要转换的字段，是在当前类声明的成员变量，所以其所在的类就是当前类。
字段引用：FieldRef 类没有直接的一个字段指向其所在的类，而是有一个所在类的名字，那么在使用字段引用(FieldRef)时这就需要以下两步解决：
- 用类加载器根据类名将对应的类加载进来
- 在从上一步加载到的类中，查找其定义的对应的字段，从而获取到真正的字段


是在方法中的 code 属性中用到的。f 方法对应的字节码为：
```
0 getstatic #2 <java/lang/System.out>
3 invokevirtual #3 <java/io/PrintStream.println>
6 return
```
其中的 getstatic 指令，后跟的索引2，指向的就是运行时常量池中的一个字段引用。那么 getstatic 指令要做的是从运行时常量池获取到一个 FieldRef，然后根据其类名加载类——System，因为 out 是类 System 中定义的一个变量，所以从 System 的 filed[] 数组中根据字段名找到对应的字段 field，
最后将该 field 压入操作数栈。



在上面的 invokevirtual 指令中，其后跟的索引3指向的是运行时常量池中的一个方法引用(MethodRef)，其方法名为println，该方法在 java/io/PrintStream 中定义，也就是 System.out 的类型。和字段引用解析类似，也是先加载其所在的类java/io/PrintStream到方法区，然后根据方法名找到对应的方法，最重要的是拿到方法中的 code 字节码，从而进行该方法的调用！

# 总结
这一节主要介绍了线程共享的运行时数据区，该数据区包含两个重要的概念：方法区和运行时常量池。同时还着重强调了两组概念：
- class 文件中的常量池 和 运行时常量池
- 类中定义的字段、方法 和 字段引用，方法引用