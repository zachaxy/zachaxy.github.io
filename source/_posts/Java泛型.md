---
title: Java泛型
date: 2018-09-23 19:55:57
tags: Java in Deep
---

> Java 泛型（generics）是 JDK 5 中引入的一个新特性, 泛型提供了编译时类型安全检测机制，该机制允许程序员在编译时检测到非法的类型。泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。

<!--more-->


`Java 中泛型的设计思想是：在编译阶段提前发现问题，而不是运行时。只要我们以这种思想去看到 Java 泛型的各种语法限制，就会感觉明朗很多。`


# 泛型在JVM 中的实现

其本质是`类型擦除`

具体可以参考[手写JVM系列-桥接方法](https://zachaxy.github.io/2017/05/09/%E6%89%8B%E5%86%99JVM%E7%B3%BB%E5%88%97-6-%E5%88%86%E6%9E%90class%E6%96%87%E4%BB%B6-%E5%B1%9E%E6%80%A7%E8%A1%A8/#%E6%A1%A5%E6%8E%A5%E6%96%B9%E6%B3%95)


# 泛型中的一些语法限制

## 1. 静态成员变量不能使用泛型
这个很好理解，静态成员变量在类加载的时候就初始化了，此时具体的泛型类型还未确定，因此定义静态成员变量无法定义。

## 2. 基本类型不能使用泛型，需要用对应的包装类型
像 int，double 等基本类型，在使用泛型时，需要转换为对应的包装类 Integer，Double 等。因为泛型参数在编译后都会替换成 Object，因此需要转换为对应的包装类，而不能是基本类型。这也引发了一个 Java 泛型设计的吐槽：对于基本类型，在泛型使用过程中，会频繁的装箱，拆箱操作，有一定的性能影响，开发者在使用时应多注意。

## 3. 无法用 instanceof 获取到 T 的类型
由于JVM 类型擦除的原因，在运行时，无法使用 instanceof 来判断 T 的类型，但是可以通过反射等方法来获取。


## 4. 可以定义泛型数组，但是不能创建泛型数组
```java
Foo<String>[] foos;  //  可以这么生命
Foo<String>[] foos = new Foo<String>[10]; // 创建则报错


// 但是也有例外，这样可以用，但是也失去了泛型的意义，强烈建议：不要这么用！
Foo<?>[] fooArray = new GenericType[10];
fooArray[0] = new Foo<Integer>();
fooArray[1] = new Foo<String>();
```


## 5. 泛型类不能继承 Throwable 及其子类

```java
// 这也定义会报错
class CustomException<T> extends Throwable {}
```

考虑如下场景：
```java
try {
    xxx
} catch (CustomException<Integer> e1) {

} catch (CustomException<String> e2) {

} catch (Exception e3) {

} 
```

由于 JVM 类型擦除的原因，e1 和 e2 其实无法区分的，因此泛型类继承 Throwable 没有实际意义，将在编译器阻断这一行为。

# 令人迷惑的通配符类型
首先定义如下类：
```java
public class GenericType<T> {
    private T t;

    public void setData(T t) {
        this.t = t;
    }

    public T getData() {
        return t;
    }
}

public class A{}
public class B extends A{}
public class C extends B{}
```

## ? extends B
```java
GenericType<? extends B> b = new GenericType<>();
// 下面的 set 方法都是不能用的
//        a.setData(new A());
//        a.setData(new B());
//        a.setData(new C());
B b1 = b.getData();
```

```java
void foo(List<? extends B> list) {
    //  list.add(new B());  // 无法添加
    //  list.add(new C());  // 无法添加
    for (int i = 0; i < list.size(); i++) {
        B b = list.get(i);
    }
}
```

在使用了 `? extends B` 通配符，变量或者集合就失去了写的能力，只能读。
这里的 `? extends B` 表示的意思是：GenericType 里面持有的数据类型被限定在 B 或者其子类，A 当然是不可以的。但是至于具体是 B 的哪个子类，不确定也不关心。但也不能随意的往里面加 B 的子类，因为 `? extends B` 是明确的某一种类型。

如果想实现添加 B 的任意子类，可以定义为 GenericType<B> 或者 List<B>


## ? super B

```java
GenericType<? super B> b = new GenericType<>();
// a.setData(new A()); 不允许
a.setData(new B());
a.setData(new C());
```

```java
void foo(List<? super B> list) {
    for (int i = 0; i < list.size(); i++) {
        Object object = list.get(i); // 其实已经不能读了,只能是 object
    }
    // list.add(new A()); 不允许
    list.add(new B());
    list.add(new C());
}
```

这里的 `? super B` 和上面的 `? extends B` 反过来了，可以写，但是不能读（读到的是 Object 类型，等同于失去了读的能力）。
但是令人疑惑的点是在写的时候，为什么不能写 B 的父类，反到是 B 的子类都可以写？

这里有一个误区：
~~认为使用了 `? super B` 就可以写 B 的父类~~

这里要整体来看 `GenericType<? super B>`  决定的依然是 GenericType f泛型的上界，只是上界具体是什么不清楚，起码 B 是安全的，B 的子类是安全。A 是 B 的父类，但是不一定安全。

# 使用场景
PECS 原则: `Producer Extends, Consumer Super`

- Producer extends: 如果我们需要一个 List 提供类型为 T 的数据(即希望从 List 中读取 T 类型的数据), 那么我们需要使用 ? extends T, 例如 List<? extends Integer>. 但是我们不能向这个 List 添加数据.

- Consumer Super: 如果我们需要一个 List 来消费 T 类型的数据(即希望将 T 类型的数据写入 List 中), 那么我们需要使用 ? super T, 例如 List<? super Integer>. 但是这个 List 不能保证从它读取的数据的类型.

如果我们既希望读取, 也希望写入, 那么我们就必须明确地声明泛型参数的类型, 例如 List<Integer>.

例子:
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
      for (int i=0; i<src.size(); i++) 
        dest.set(i,src.get(i)); 
  } 
```
上面的例子是一个拷贝数据的代码, src 是 List<? extends T> 类型的, 因此它可以读取出 T 类型的数据(读取的数据类型是 T 或是 T 的子类, 但是我们不能确切的知道它是什么类型, 唯一能确定的是读取的类型 is instance of T), dest 是 List<? super T> 类型的, 因此它可以写入 T 类型或其子类的数据.