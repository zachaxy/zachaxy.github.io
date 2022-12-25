---
title: Android中的序列化
date: 2017-08-08 11:21:17
tags: Android Framework
---

> 当我们启动一个新的 Activity,或者创建一个 Fragment 想要携带参数时,尤其是要传递一个对象时,需要将该对象放到 Bundle 中. Bundle 提供的 api 能力,有两种可选项,要么该对象实现了 Serializable 接口,要么该对象实现了 Parcelable. Serializable 是 Java 提供的序列化机制,既然已经有了成熟的序列化机制,为什么 Android 还要额外提供一种序列化机制(Parcelable)呢? 本文将对 Android 中的序列化机制做一个讲解.

<!--more-->

# Serializable 的使用
```java
public class SerializableDeveloper implements Serializable {
    String name;
    int yearsOfExperience;
    List<Skill> skillList;
    float favoriteFloat;

    static class Skill implements Serializable {
        String name;
        boolean programmingRelated;
    }
}
```
Serializable 的使用方法非常简单,你只需要让目标类实现Serializable接口,同时目标类中的成员变量也需要实现Serializable接口.
但是随之而来的是一些性能上的问题, Serializable 机制内部会使用反射,这样会降低序列化的效率,同时也会创建很多临时变量,引发频繁的 GC.
这种性能的损耗,尤其是在移动设备上是尤为显著的. 
因此,Android 设计了另一种序列化机制来避免改问题.



# Parcelable

```java
class ParcelableDeveloper implements Parcelable {
    String name;
    int yearsOfExperience;
    List<Skill> skillSet;
    float favoriteFloat;

    ParcelableDeveloper(Parcel in) {
        this.name = in.readString();
        this.yearsOfExperience = in.readInt();
        this.skillSet = new ArrayList<Skill>();
        in.readTypedList(skillSet, Skill.CREATOR);
        this.favoriteFloat = in.readFloat();
    }

    void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(yearsOfExperience);
        dest.writeTypedList(skillSet);
        dest.writeFloat(favoriteFloat);
    }

    int describeContents() {
        return 0;
    }


    static final Parcelable.Creator<ParcelableDeveloper> CREATOR
            = new Parcelable.Creator<ParcelableDeveloper>() {

        ParcelableDeveloper createFromParcel(Parcel in) {
            return new ParcelableDeveloper(in);
        }

        ParcelableDeveloper[] newArray(int size) {
            return new ParcelableDeveloper[size];
        }
    };

    static class Skill implements Parcelable {
        String name;
        boolean programmingRelated;

        Skill(Parcel in) {
            this.name = in.readString();
            this.programmingRelated = (in.readInt() == 1);
        }

        @Override
        void writeToParcel(Parcel dest, int flags) {
            dest.writeString(name);
            dest.writeInt(programmingRelated ? 1 : 0);
        }

        static final Parcelable.Creator<Skill> CREATOR
            = new Parcelable.Creator<Skill>() {

            Skill createFromParcel(Parcel in) {
                return new Skill(in);
            }

            Skill[] newArray(int size) {
                return new Skill[size];
            }
        };

        @Override
        int describeContents() {
            return 0;
        }
    }
}
```

Parcelable 的使用方法略微复杂,在实现 Parcelable 的同时,还要实现一些列的模板代码(好在 Android Studio 自带的模板机制可以简化这一流程), Parcelable 的序列化过程没有使用反射,而是在模板代码中由使用者自己定义.


# Serializable VS Parcelable 序列化速度对比
测试方法:
1. 在 Activity 中创建 Bundle, 将序列化的对象分别以 Serializable 和 Parcelable 的方式传入 bundle 中, 再取出来
1. 执行上述操作 1000 次, 作为一组
1. 分别执行 10 组,取平均值,记录传输速度,内存占用率,CPU 使用率
1. 测试对象为上述的: SerializableDeveloper, ParcelableDeveloper

测试设备:
1. LG Nexus 4 - Android 4.2.2 
1. Samsung Nexus 10 - Android 4.2.2
1. HTC Desire Z - Android 2.3.3


测试结果:

![速度对比](https://raw.githubusercontent.com/zachaxy/pic-repository/master/imgparcelable-vs-serializable-e1366334109758.png)

# 使用 intent 传递数据时的大小限制
通过 intent 的 bundle 的源码可以看到它们都是实现了 Parcelable ，其实就是通过序列化来实现通信的。Parcelable 的底层使用了 Parcel 机制。其传输过程实际上是使用了 binder 机制，binder 机制会将 Parcel序列化的数据写入到一个~共享内存~中，读取时也是 binder 从共享内存中读出字节流，然后 Parcel 反序列化后使用。这就是 Intent 或 Bundle 能够在 activity或者跨进程通信的原理。

![parcel传输原理](https://raw.githubusercontent.com/zachaxy/pic-repository/master/img20210526130417934.png)

简单来说，我们上面提到了 Parcel 机制使用了一个共享内存，这个共享内存就叫 Binder transaction buffer，这块内存有一个大小限制，目前是 1MB，而且共用的，当超过了这个大小就会报错。




# Activity之间传递较大数据的方式

1. 限制传递数据量
1. 改变数据传输方式（参见Activity之间传递数据的方式）
1. 全局变量,外部都能访问到:
    1. 静态static
    1. 单例
    1. Application
    1. 使用容器的单例模式
1. 持久化
    1. sp
    1. sqlite

# 参考链接
- [Android 使用 intent 传递数据时的大小限制](https://blog.csdn.net/u011033906/article/details/89316543)
- [Parcelable vs Serializable](http://www.developerphil.com/parcelable-vs-serializable/)
- [解析Serializable原理](https://juejin.cn/post/6844904049997774856)
    