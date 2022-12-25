---
title: JVM内存管理
date: 2018-06-02 22:51:59
tags: JVM
---
jdk 中各种工具的使用：
javap


解释型语言 vs JIT
C++解释器，


hotspot：到底是什么？ 在服务端执行，循环超过 1w 次，会进行优化；直接翻译成机器码，汇编；
坏处就是编译慢，预热， 

jvm 发展史
https://rumenz.com/rumenbiji/java-jvm-history.html   
https://rumenz.com/rumenbiji/java-jvm-history.html

直接内存，居然还有 JVM 没有虚拟化的内存空间，jvm 也可以用



-Xss 线程堆栈大小，不是栈帧的数量，这个栈难道是一开始就设计好的？ 平台相关，默认基本上为 1M 可以调整
一个可以优化的点是： 如果当前是多线程系统，维持在 500 个线程；如果当前仅剩 200M 内存，那么可以通过动态设置 Xss 小一点，保证那 500 个线程都能运行起来


栈帧到底包含什么？




javap -v xxx.class  查看字节码


jvm 操作不了线程

Unsafe 类，宝藏

JHSDB 工具查看内存

jps 命令是干什么的，查看 java 进程吗？

垃圾回收：15 次，因为 jvm 用了 4 bits 二进制 1111 作为变量；垃圾回收一次，还没有被回收掉，age + 1；  age 达到 15， 晋为 老年代