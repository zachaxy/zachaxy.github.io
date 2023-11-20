---
title: kotlin-Nothing关键字
date: 2023-11-11 15:42:22
tags: [编程语言,kotlin]
---
是所有类的子类，

编译后还是 Void



kotlin 集合和 java 集合的关系

java 使用不可变集合： unmodify， 1.9 之后， list.of

为什么引入不可变集合
- 防止其它使用方修改集合数据
- 多线程条件下，无线程安全问题
- 不存在扩容问题，节省内存消耗


互操作存在的问题


操作符，会创建新的列表；

list.map.filter

list.asSequence.map.filter