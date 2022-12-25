---
title: 快速绘制UML图
date: 2019-10-01 00:13:25
tags: 工具
---

之前介绍了一些[UML图基础](https://zachaxy.github.io/2018/04/20/uml%E5%9B%BE%E5%9F%BA%E7%A1%80/)的概念,这次推荐一个快速绘制UML图的工具

<!-- more -->

# 绘制工具
在经历了一番折腾后,最终还是选择了 [PlantUML](http://plantuml.com/zh/),原因是看重其使用特有的语言书写接口,类,并自动生成UML. 同时,自己也在使用 VSCode ,发现 PlantUML 在 VSCode 上提供了插件,于是直接在 VSCode 上安装了 PlantUML的插件.
同时,要实时预览,还要安装 `graphviz`, 如果是 Mac,那么直接使用 homebrew 安装即可: `brew install graphviz`, 安装完成很后,使用 `dot -v` 查看是否安装成功

在 VSCode 中,使用 `alt` + `d` 进行实时预览

# 常用语法

## 类声明
- 接口声明: interface 接口名
- 抽象类声明: abstract class 类名
- 类声明: class 类名
- 注解类声明: annotation 注解名
- 枚举类声明: enum 枚举类名

## 属性/方法修饰符
|关系|符号|
|:--:|:--:|
|public|+|
|protected|#|
|default|~|
|private|-|


## PlantUML用下面的符号来表示类之间的关系:

|关系|符号|说明|
|:--:|:--:|:--:|
|实现| `<|..` |Realization, 接口的实现,带空心三角型的虚线表示|
|泛化| `<|–-` |Generalization, 类的继承,带空心三角型的直线表示|
|依赖| `<..` |Dependency, 带箭头的虚线表示|
|组合| `*–-` |Composition, 带实心的菱形的直线表示|
|聚合| `o–-` |Aggregation, 带空心菱形的直线表示|
|关联| `<–` |Association|
|单向关联| `<--` |Association,带箭头的实线表示|
|双向关联| `--` |直线表示|
|多重性关联| `“1..*”<--"0..*"` |关联直线上用一个数字或者一个数字的范围表示|

## 一个例子

```
@startuml

interface IUser

abstract class BaseUser
class User
class ShadowUser

IUser <|.. BaseUser
IUser <|.. ShadowUser
BaseUser <|-- User

interface IUserCenter {
    + IUser getCurrentUser()
    + boolean isLogin()
    + void queryUserProfile(int uid, string sid)
    + void search(int uid, string sid)
    + IUpdater updater(int uid, string sid)
}

class UserCenter {
    - HostUserManager
    - GuestUserManager
}

IUserCenter <|.. UserCenter


interface IUserManager {
    void getUser(int uid, string sid)
    void getProfile(int uid, string sid)
}

class HostUserManager {
    - HostUserCache
    - HostUpdater
}
class GuestUserManager {
    - GuestUserCache
    - GuestUpdater
}

IUserManager <|..HostUserManager
IUserManager <|..GuestUserManager

interface IUserCache {
    + void cacheUser(IUser user)
    + void deleteUser(int uid)
    + boolean hasUser(int uid)
}


class HostUserCache {
    - SharedPreference sp
}
class GuestUserCache {
    - HashMap cache
}
IUserCache <|..HostUserCache
IUserCache <|..GuestUserCache


interface IUpdater {
    + void setXXX(var xxx)
    + void update()
}
class HostUpdater
class GuestUpdater
IUpdater <|..HostUpdater
IUpdater <|..GuestUpdater

note top of IUpdater:主态update方法之后,需要拉取下User,客态update方法之后,直接设置回UserCache即可

UserCenter <-- HostUserManager
UserCenter <-- GuestUserManager

HostUserManager <-- HostUserCache
HostUserManager <-- HostUpdater

GuestUserManager <-- GuestUserCache
GuestUserManager <-- GuestUpdater


note as N1 #red
IUserCenter 作为User相关操作对外暴露的唯一对象
UserCache,UserManager 均为内部实现
end note

@enduml
```

 # 参考连接
- [对象图](http://plantuml.com/zh/object-diagram)
- [用VS Code画uml](https://blog.csdn.net/qq_26819733/article/details/84895850)
- [使用PlantUML快速绘图](https://blog.csdn.net/zxc123e/article/details/71837923)
- [UML Class Diagram Tutorial](https://www.youtube.com/watch?v=UI6lqHOVHic)