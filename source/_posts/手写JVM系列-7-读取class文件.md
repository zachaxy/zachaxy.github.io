---
title: 手写JVM系列(7)-读取class文件
date: 2017-05-10 16:21:24
tags: JVM
---

> 通过前面几节的讲解，我们已经基本了解了 class 文件的结构,每个字段代表什么含义,占多少字节,那么明确了 Class 文件的规则,接下来就可以读取 class 文件了，本节的代码均在[项目](https://github.com/zachaxy/JVM)的 classfile 包下。

------------------



# Class 文件的结构

这里再次贴出 class 文件的结构描述：

```java
ClassFile {
  u4 magic;	//魔数
  u2 minor_version;	//次版本号
  u2 major_version;	//主版本号
  u2 constant_pool_count;	//常量池大小
  cp_info constant_pool[constant_pool_count-1]; //常量池
  u2 access_flags;	//类访问标志,表明 class 文件定义的是类还是接口，访问级别是 public 还是 private，等
  u2 this_class;	//
  u2 super_class;	//
  u2 interfaces_count;	//本类实现的接口数量
  u2 interfaces[interfaces_count];	//实现的接口,存放在数组中
  u2 fields_count;		//本来中含有字段数
  field_info fields[fields_count];	//数组中存放这各个字段
  u2 methods_count;		//本类中含有的方法数
  method_info methods[methods_count];	//数组中存放着各个方法
  u2 attributes_count;			//本类中含有的属性数量;
  attribute_info attributes[attributes_count];	//数组中存放着各个属性
}
```



# ClassFile 类

根据上面的 class 结构类型，我们自己定义的类 ClassFile 也呼之欲出,每个字段和上述 class 结构几乎是一样的。类中的一个成员变量定义如下

```java
public class ClassFile {
    
    int minorVersion;
    int majorVersion;
    ConstantPool constantPool;
    int accessFlags;
    int thisClass;
    int superClass;
    int[] interfaces;
    MemberInfo[] fields;
    MemberInfo[] methods;
    AttributeInfo[] attributes;


    public ClassFile(byte[] classData) {
        ClassReader reader = new ClassReader(classData);
        read(reader);
    }

    void read(ClassReader reader) {
        readAndCheckMagic(reader);  //验证魔数
        readAndCheckVersion(reader);	//校验版本
        constantPool = new ConstantPool(reader);	//创建常量池
        accessFlags = reader.readUint16();		//获取类访问标志
        thisClass = reader.readUint16();		//
        superClass = reader.readUint16();		//
        interfaces = reader.readUint16s();		//
        fields = MemberInfo.readMembers(reader, constantPool);	//
        methods = MemberInfo.readMembers(reader, constantPool);	//
        attributes = AttributeInfo.readAttributes(reader, constantPool);	//
    }
}
```

可以很清晰的看到,我们定义的 ClassFile 文件,可以看到成员变量和 JVM 中关于 class 文件的描述是一致的,只不过这里为了方便编码,统一用 int 类型来保存 u1,u2 和 u4 类型的值.



# Class 文件字节码读取辅助类

接下来要解决的问题是每个字段所占的字节数不同,所以这里我们需要若干方法根据字节数读取相应的字节,所有有创建了一个`ClassReader`类,并令该类持有 class 字节码，并且在该类中保存一个 index，表明现在是从哪个字节开始读.并且提供了读取 1 字节,2 字节,4 字节,8 字节等方法,来满足`ClassFile`类中个字段对应的字节数的需求。

```java
public class ClassReader {

    byte[] data;
    int index = 0;

    public ClassReader(byte[] data) {
        this.data = data;
    }

    // u1
    public byte readUint8() {
        byte res = data[index++];
        return res;
    }


    // u2 这里是读取一个无符号的 16 位整,java 中没有,只能用 int 来代替吧;
    public int readUint16() {
        byte[] res = new byte[2];
        res[0] = data[index++];
        res[1] = data[index++];
        return ByteUtils.bytesToU16(res);
    }

    // u4
    public byte[] readUint32() {
        byte[] res = new byte[4];
        res[0] = data[index++];
        res[1] = data[index++];
        res[2] = data[index++];
        res[3] = data[index++];
//        return ByteUtils.bytesToU32(res);  //如果需要转换的话,自行调用 ByteUtils 中的方法;
        return res;
    }

    public byte[] readUint64() {
        byte[] res = new byte[8];
        res[0] = data[index++];
        res[1] = data[index++];
        res[2] = data[index++];
        res[3] = data[index++];
        res[4] = data[index++];
        res[5] = data[index++];
        res[6] = data[index++];
        res[7] = data[index++];
        return res;
    }

    public int[] readUint16s() {
        int n = readUint16();
        int[] data = new int[n];
        for (int i = 0; i < n; i++) {
            data[i] = readUint16();
        }
        return data;
    }

    public byte[] readBytes(int n) {
        byte[] res = new byte[n];
        for (int i = 0; i < n; i++) {
            res[i] = data[index++];
        }
        return res;
    }
    
}
```

在有了 ClassReader 这样的工具类之后,我们就可以在 ClassFile 中根据不同的字段，使用不同的 ClassReader#readXXX() 方法,来初始化成员变量,这里定义了一个 ClassFile#read() 方法来完成 ClassFile 类的初始化成员变量的任务。



# 简单字段的实现

对于 ClassFile 的字段，除了常量和属性两个区域，其它的字段都可以很容易根据其占用的字节长度读出来，这对于编码来说并没有什么难度。在 ClassFile#read 方法中实现了所有字段的读取。

```java
void read(ClassReader reader) {
    readAndCheckMagic(reader);
    readAndCheckVersion(reader);
    constantPool = new ConstantPool(reader);
    accessFlags = reader.readUint16();
    thisClass = reader.readUint16();
    superClass = reader.readUint16();
    interfaces = reader.readUint16s();
    fields = MemberInfo.readMembers(reader, constantPool);
    methods = MemberInfo.readMembers(reader, constantPool);
    attributes = AttributeInfo.readAttributes(reader, constantPool);
}
```

但是对于常量和属性，因为其各自又包含了许多种类，所以需要针对不同的常量，不同的属性进行不同的读取。接下来分别介绍如何实现常量和属性的读取。



# 常量的实现

这里需要定义一个常量的抽象类 ConstantInfo，表示一个常量 item，具体的常量由其子类实现，这里对外提供一个统一的接口来根据不同的 tag，创建不同的具体常量实现类，以完成常量池的初始化。

```java
private static ConstantInfo create(int tag, ConstantPool constantPool) {
    switch (tag) {
        case CONSTANT_Integer:
            return new ConstantIntegerInfo();
        case CONSTANT_Float:
            return new ConstantFloatInfo();
        case CONSTANT_Long:
            return new ConstantLongInfo();
        case CONSTANT_Double:
            return new ConstantDoubleInfo();
        case CONSTANT_Utf8:
            return new ConstantUtf8Info();
        case CONSTANT_String:
            return new ConstantStringInfo(constantPool);
        case CONSTANT_Class:
            return new ConstantClassInfo(constantPool);
        case CONSTANT_Fieldref:
            return new ConstantMemberRefInfo(constantPool);
        case CONSTANT_Methodref:
            return new ConstantMemberRefInfo(constantPool);
        case CONSTANT_InterfaceMethodref:
            return new ConstantMemberRefInfo(constantPool);
        case CONSTANT_NameAndType:
            return new ConstantNameAndTypeInfo();
        // TODO: 2017/5/3 0003 下面三个类还未编码; 
        case CONSTANT_MethodType:
            return new ConstantMethodTypeInfo();
        case CONSTANT_MethodHandle:
            return new ConstantMethodHandleInfo();
        case CONSTANT_InvokeDynamic:
            return new ConstantInvokeDynamicInfo();
        default:
            throw new RuntimeException("java.lang.ClassFormatError: constant pool tag!");
    }
}
```



并且提供一个抽象方法，供子类实现，因为每种常量所占的字节数并不相同。

```java
abstract void readInfo(ClassReader reader)
```

而对于各自具体的常量，需要根据各自常量的结构来读取，其结构已经在[手写 JVM 系列(5)-分析 class 文件-常量池](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-5-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B8%B8%E9%87%8F%E6%B1%A0/)中进行了详细的介绍。具体实现请参照[项目源码](https://github.com/zachaxy/JVM)



# 常量池的实现

有了上面各个常量的具体实现，那么接下来我们就可以构建常量池了。常量池其实就是本 class 文件中所有常量的集合。

因为常量池是根据索引来访问的，因此我们也很自然的想到用数组来表示常量池，数组类型是上面定义的常量类型，注意索引从 1 开始，0 是无效索引。常量池的初始化时在构造方法中，通过 ConstantInfo 提供的 readConstantInfo 静态方法，读取一字节 tag，根据 tag 创建不同的常量实现类，并添加到常量池数组中。



```java
public class RuntimeConstantPool {
    ConstantInfo[] infos;  //保存类文件常量池中的所有常量,常量分为多种类型,基本类型都有对应的常量,以及字符串等;(简言之,这就是常量池的抽象)

    int constantPoolCount; //class 文件中常量池中的常量数量

    public ConstantPool(ClassReader reader) {
        /*读出常量池的大小;接下来根据这个大小,生成常量信息数组;
        注意:
        1. 表头给出的常量池大小比实际大 1,所以这样的话,虽然可能生成了这么大的,但是 0 不使用,直接从 1 开始;
        2. 有效的常量池索引是 1~n–1。0 是无效索引，表示不指向任何常量
        3. CONSTANT_Long_info 和 CONSTANT_Double_info 各占两个位置。
           也就是说，如果常量池中存在这两种常量，实际的常量数量比 n–1 还要少，而且 1~n–1 的某些数也会变成无效索引。
        */
        constantPoolCount = reader.readUint16();
        infos = new ConstantInfo[constantPoolCount];
        for (int i = 1; i < constantPoolCount; i++) {
            infos[i] = ConstantInfo.readConstantInfo(reader, this);
            if ((infos[i] instanceof ConstantLongInfo) || (infos[i] instanceof ConstantDoubleInfo)) {
                i++;
            }
        }

    }
  ......
}
```
# 属性的实现

对于属性的编码，和常量池的编码思路是相似的，其实这里称为“属性池”更为贴切。因为他是各种属性的集合。而 class 文件本身，方法表集合和字段表集合中均持有属性表。



同样，提供一个抽象类来表示一个属性，其定义如下：

```java
public abstract class AttributeInfo {


    abstract void readInfo(ClassReader reader);

    //读取单个属性
    private static AttributeInfo readAttribute(ClassReader reader, ConstantPool constantPool) {
        int attrNameIndex = reader.readUint16();
        String attrName = constantPool.getUtf8(attrNameIndex);
        int attrLen = ByteUtils.byteToInt32(reader.readUint32());
        AttributeInfo attrInfo = create(attrName, attrLen, constantPool);
        attrInfo.readInfo(reader);
        return attrInfo;
    }

    //读取属性表;这个和 ConstantPool 中的方法类似,一般都是一下全部读取出来,不会只读一个
    public static AttributeInfo[] readAttributes(ClassReader reader, ConstantPool constantPool) {
        int attributesCount = reader.readUint16();
        AttributeInfo[] attributes = new AttributeInfo[attributesCount];
        for (int i = 0; i < attributesCount; i++) {
            attributes[i] = readAttribute(reader, constantPool);
        }
        return attributes;
    }

    //Java 虚拟机规范预定义了 23 种属性，先解析其中的 8 种
    /*
    23 种预定义属性可以分为三组。
    第一组属性是实现 Java 虚拟机所必需的，共有 5 种；
    第二组属性是 Java 类库所必需的，共有 12 种；
    第三组属性主要提供给工具使用，共有 6 种。第三组属性是可选的，也就是说可以不出现在 class 文件中。
    (如果 class 文件中存在第三组属性，Java 虚拟机实现或者 Java 类库也是可以利用它们的，比如使用 LineNumberTable 属性在异常堆栈中显示行号。)
     */
    private static AttributeInfo create(String attrName, int attrLen, ConstantPool constantPool) {
        if (attrName.equals("Code")) {
            return new CodeAttribute(constantPool);
        }else if (attrName.equals("ConstantValue")){
            return new ConstantValueAttribute();
        }else if (attrName.equals("Deprecated")){
            return new DeprecatedAttribute();
        }else if (attrName.equals("Exceptions")){
            return new ExceptionsAttribute();
        }else if (attrName.equals("LineNumberTable")){
            return new LineNumberTableAttribute();
        }else if (attrName.equals("LocalVariableTable")){
            return new LocalVariableTableAttribute();
        }else if (attrName.equals("SourceFile")){
            return new SourceFileAttribute(constantPool);
        }else if (attrName.equals("Synthetic")){
            return new SyntheticAttribute();
        } else {
            return new UnparsedAttribute(attrName, attrLen);
        }

    }
}
```

其内部定义了抽象方法 readInfo，供各具体的属性类去读取相应的数据。而对外，提供了一个 readAttributes 的方法，来返回当前 方法表集合或者字段表集合中的的属性集合。与常量池不同的是：常量是根据不同的 tag（代表一个整数）来区分不同的常量，而属性是根据不同的 name（字符串）来区分不同的属性，所以创建属性的方法 AttributeInfo#create 方法。



而对于各自具体的属性，需要根据各自属性的结构来读取，其结构已经在[手写 JVM 系列(6)-分析 class 文件-属性表](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/)中进行了详细的介绍。具体实现请参照[项目源码](https://github.com/zachaxy/JVM)
