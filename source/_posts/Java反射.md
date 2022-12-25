---
title: Java反射
date: 2018-09-27 08:40:08
tags: Java in Deep
---

> 反射是 Java 语言中的高级功能。它允许 Java 程序在运行中拿到一个类的所有信息（方法名称，字段名称），并且动态执行对象的方法或修改字段的值。

<!-- more -->

# 获取 Class 对象的三种方式
1. 通过类名获取：`Class clazz = String.class;`
1. 通过对象获取：`Class clazz = "str".getClass();`
1. 通过全类名获取：`Class clazz  = Class.forName("java.lang.String");`
1. 通过 ClassLoader + 全类名获取：`Class clazz = classLoader.loadClass("java.lang.String");`


# 常用方法
通过上述获取到的 clazz 对象，进而可以获取对应类的具体信息。下面 api 中，会频繁出现 getXXX 和 getDeclaredXXX 的方法。其中前者表示获取的是当前类中 public 的 xxx，而后者表示的是获取当前类中含 protect，default，private 的 xxx。
## 获取构造方法
- 获取当前类（不包含父类）的所有 public 构造方法：
```java
Constructor[] constructors = clazz.getConstructors();
```
- 根据参数类型返回指定构造方法：
```
Constructor constructor = clazz.getConstructor(Class<?>... parameterTypes);
```
- 获取当前类（不包含父类）的所有构造方法（含private）：
```
Constructor[] constructors = clazz.getDeclaredConstructors();
```
- 根据参数类型返回指定构造方法：
```
Constructor constructor = clazz.getDeclaredConstructor(Class<?>... parameterTypes);
```

## 创建新对象
- 无参构造方法，且是 public，可以直接：
```
Object obj = clazz.newInstance();
```
- 有参构造方法，需要通过上一步获取到的 constructor
```
Object obj = constructor.newInstance(Object... args);
```

注意：如果 constructor 非 public，需要先调用 `constructor.setAccessible(true);`
再创建对象，否则会抛出 IllegalAccessException。
下面 method 和 filed 在使用时，也有类似问题。


## 获取方法
- 获取public方法
```java
Method[] methods = clazz.getMethods();
```
- 获取 public 指定方法
```java
Method method = clazz.getMethod(String name, Class<?>... parameterTypes)
```
- 获取所有方法
```java
Method[] methods = clazz.getDeclaredMethods();
```
- 获取所有方法中的指定方法
```java
Method method = clazz.getDeclaredMethod(String name, Class<?>... parameterTypes)
```

## 执行方法
```
// 如果 method 为非 public，则需要执行如下：
method.setAccessible(true);
// 指定方法，第一个参数是传入的对象，如果是静态方法，则传 null
method.invoke(Object obj, Object... args) 
```

## 获取字段
```java
Field[] fields = clazz.getFields();
Field field = clazz.getField(String name);
Field[] fields = clazz.getDeclaredFields();
Field field = clazz.getDeclaredField(String name);
```

## 读、写 field
```
// 如果 field 为非 public，则需要执行如下：
field.setAccessible(true);

// 读 
Object value = field.get(obj);
// 写
field.set(obj, arg);
```

# 创建数组
```
String[] array = (String[])Array.newInstance(String.class, 10);
```

# 获取泛型类型
![Type类关系图](https://raw.githubusercontent.com/zachaxy/pic-repository/master/imgJava_Type%20%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

Type：用来表示某个字段的类型
- Class：普通类的类型
- TypeVariable：泛型类型的变量
- ParameterizedType：带泛型的类型： eg： `List<String> list`
- GenericArrayType：泛型类型的数组： eg `List<String>[] list`
- WildcardType：**通配符**泛型类型： eg：`List<? extends Number> list`

具体看下面 demo 代码：
```java
public class TypeDemo<T extends Comparable & Serializable> {
    String str;
    T t;
    Map<String, Integer> map;
    List<String>[] listsArray;
    List<? extends Number> numberList;

    public static void main(String[] args) {
        try {
            demo4Normal();
            demo4TypeVariable();
            demo4ParameterizedType();
            demo4GenericArrayType();
            demo4WildcardType();

        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
    }

    private static void demo4Normal() throws NoSuchFieldException {
        System.out.println("===demo4Normal===");
        Field field = TypeDemo.class.getDeclaredField("str");
        System.out.println("GenericType: " + field.getGenericType());
        System.out.println("GenericType Class: " + field.getGenericType().getClass());
    }

    private static void demo4TypeVariable() throws NoSuchFieldException {
        System.out.println("===demo4TypeVariable===");
        Field field = TypeDemo.class.getDeclaredField("t");
        System.out.println("GenericType: " + field.getGenericType());
        System.out.println("GenericType Class: " + field.getGenericType().getClass());
        TypeVariable type = (TypeVariable) field.getGenericType();
        // T
        System.out.println("TypeVariable name: " + type.getName());


        // 获取声明该泛型变量的 owner 的 class 对象: -->  class zx.reflect.TypeDemo
        System.out.println("GenericDeclaration: " + type.getGenericDeclaration());

        /**
         * java.lang.Comparable
         * java.io.Serializable
         */
        for (Type bound : type.getBounds()) {
            System.out.println(bound.getTypeName());
        }
    }

    private static void demo4ParameterizedType() throws NoSuchFieldException {
        System.out.println("===demo4ParameterizedType===");
        Field field = TypeDemo.class.getDeclaredField("map");
        System.out.println("GenericType: " + field.getGenericType());
        System.out.println("GenericType Class: " + field.getGenericType().getClass());
        ParameterizedType parameterizedType = (ParameterizedType) field.getGenericType();
        System.out.println("RawType: " + parameterizedType.getRawType());
        System.out.println("OwnerType: " + parameterizedType.getOwnerType());
        for (Type actualTypeArgument : parameterizedType.getActualTypeArguments()) {
            System.out.println("ActualTypeArgument: " + actualTypeArgument);
        }
    }

    private static void demo4GenericArrayType() throws NoSuchFieldException {
        System.out.println("===demo4GenericArrayType===");
        Field field = TypeDemo.class.getDeclaredField("listsArray");
        System.out.println("GenericType: " + field.getGenericType());
        System.out.println("GenericType Class: " + field.getGenericType().getClass());
        GenericArrayType genericArrayType = (GenericArrayType) field.getGenericType();
        System.out.println("GenericComponentType: " + genericArrayType.getGenericComponentType());
    }

    private static void demo4WildcardType() throws NoSuchFieldException {
        System.out.println("===demo4WildcardType===");
        Field field = TypeDemo.class.getDeclaredField("numberList");
        System.out.println("GenericType: " + field.getGenericType());
        System.out.println("GenericType Class: " + field.getGenericType().getClass());
        ParameterizedType parameterizedType = (ParameterizedType) field.getGenericType();
        // 这里确定只有一个,直接取数组下标 0
        Type type = parameterizedType.getActualTypeArguments()[0];
        System.out.println("ActualTypeArgument GenericType: " + type);
        System.out.println("ActualTypeArgument GenericType Class: " +type.getClass());
        WildcardType wildcardType = (WildcardType) type;
        for (Type lowerBound : wildcardType.getLowerBounds()) {
            System.out.println("lowerBound: " + lowerBound);
        }
        for (Type upperBound : wildcardType.getUpperBounds()) {
            System.out.println("upperBound: " + upperBound);
        }
    }
}
```


# 获取注解类型
```

```


# 使用反射可以获取private字段的值，类的封装还有意义吗
正常情况下，我们总是通过符合 Java 语法规范的代码来访问合法的方法或字段，编译器会根据public、protected，private决定是否允许访问字段，这样就达到了数据封装的目的。

而反射是一种非常规的用法，使用反射，首先代码非常繁琐，其次，它更多地是给工具或者底层框架来使用，目的是在不知道目标实例任何信息的情况下，获取特定字段的值。

此外，**setAccessible(true)可能会失败**。如果JVM运行期存在SecurityManager，那么它会根据规则进行检查，有可能阻止setAccessible(true)。例如，某个SecurityManager可能不允许对java和javax开头的package的类调用setAccessible(true)，这样可以保证JVM核心库的安全。

# gson中 TypeToken 是如何获取的
todo:单独放在一篇里面，篇幅太多了。
```java
public class GenericType<T> {
    private T t;

    public T getT() {
        return t;
    }

    public void setT(T t) {
        this.t = t;
    }
}
```
对应的字节码如下：

```
// class version 52.0 (52)
// access flags 0x21
// signature <T:Ljava/lang/Object;>Ljava/lang/Object;
// declaration: zx/generic/GenericType<T>
public class zx/generic/GenericType {

    // compiled from: GenericType.java
  
    // access flags 0x2
    // signature TT;
    // declaration: T
    private Ljava/lang/Object; t
  
    // access flags 0x1
    public <init>()V
     L0
      LINENUMBER 7 L0
      ALOAD 0
      INVOKESPECIAL java/lang/Object.<init> ()V
      RETURN
     L1
      LOCALVARIABLE this Lzx/generic/GenericType; L0 L1 0
      // signature Lzx/generic/GenericType<TT;>;
      // declaration: zx.generic.GenericType<T>
      MAXSTACK = 1
      MAXLOCALS = 1
  
    // access flags 0x1
    // signature ()TT;
    // declaration: T getT()
    public getT()Ljava/lang/Object;
     L0
      LINENUMBER 11 L0
      ALOAD 0
      GETFIELD zx/generic/GenericType.t : Ljava/lang/Object;
      ARETURN
     L1
      LOCALVARIABLE this Lzx/generic/GenericType; L0 L1 0
      // signature Lzx/generic/GenericType<TT;>;
      // declaration: zx.generic.GenericType<T>
      MAXSTACK = 1
      MAXLOCALS = 1
  
    // access flags 0x1
    // signature (TT;)V
    // declaration: void setT(T)
    public setT(Ljava/lang/Object;)V
     L0
      LINENUMBER 15 L0
      ALOAD 0
      ALOAD 1
      PUTFIELD zx/generic/GenericType.t : Ljava/lang/Object;
     L1
      LINENUMBER 16 L1
      RETURN
     L2
      LOCALVARIABLE this Lzx/generic/GenericType; L0 L2 0
      // signature Lzx/generic/GenericType<TT;>;
      // declaration: zx.generic.GenericType<T>
      LOCALVARIABLE t Ljava/lang/Object; L0 L2 1
      // signature TT;
      // declaration: T
      MAXSTACK = 2
      MAXLOCALS = 2
  }
```





```java
public class SubGenericType extends GenericType<String> {}
```

对应的字节码如下：
```
// class version 52.0 (52)
// access flags 0x21
// signature Lzx/generic/GenericType<Ljava/lang/String;>;
// declaration: zx/generic/SubGenericType extends zx.generic.GenericType<java.lang.String>
public class zx/generic/SubGenericType extends zx/generic/GenericType  {

  // compiled from: SubGenericType.java

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 7 L0
    ALOAD 0
    INVOKESPECIAL zx/generic/GenericType.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lzx/generic/SubGenericType; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1
}
```


# 反射对性能的影响
这里先给出结论：
- 通过 Class 的对象来获取 Method，Filed 等行为，是非常耗时的方法，如果非要使用反射，且会频繁的对某个固定的 Class 对象进行反射的操作，应该在第一次获取到 Method，Filed时，缓存起来。
- setAccessible(true) 方法并非简单的赋值，而是一个较为耗时的方法，如果我们确定其为 public 方法，应该避免调用 setAccessible 方法。

todo：待 benchmark 测试

# JVM中反射的实现机制
可以参考自动动手写 JVM 中，[JVM中反射的实现机制](https://zachaxy.github.io/2018/01/18/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-16-%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6%E7%AE%80%E4%BB%8B/) 对反射机制的实现。

# 常用的反射库
- [reflections](https://github.com/ronmamo/reflections)
- [jOOR](https://github.com/jOOQ/jOOR)
- [Mirror](https://github.com/vidageek/mirror/)
