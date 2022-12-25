---
title: Windows配置JNI开发环境
date: 2018-01-10 20:06:54
tags: JNI
---

# JNI 简介
JNI：Java Native Interface。
当 JDK 所提供的 API 无法满足当前应用的需求时,就要调用 native interface。我们知道 Java 程序跨平台是有前提的，其并不是直接运行时操作系统上的，而是运行在 Java 虚拟机上的，我们平时调用的 JDK 中的 api 是基于 JVM 实现的,但是有一些情况下,这些 api 不能满足我们对性能的要求，那么就需要调用更为底层的语言来解决效率问题。所谓底层语言，一般是说 c/c++，这类语言是可以直接和操作系统的内存做交互的，所以在效率上更加高效。

<!--more-->

为了能都调用 c/c++这些底层的语言的接口，Java 提供了 JNI 机制,用来调用动态链接库。我们只需要在对应平台上将 c/c++打包成动态链接库，上层的 Java 就可以通过 JNI 调用动态链接库中的接口了。
不同操作系统上，动态链接库的表现形式也是不同的，

- window：dll 文件
- linux：so 文件
- mac：framework 文件

原始的 c 文件都是相同的，只不过由于平台不同，其打包成的动态链接库文件的格式也是不同的。

我们的目的是尽快熟悉 JNI 开发流程，而并不关心动态链接库的内部，同时，我们也不想花大把时间在配置上，所以在 window 平台上，下面所提供的方法是最简单省事的，记住我们的目标学习 JNI，不要在其它事情上花费太多时间。其实在 linux 平台上，都不需要搭建这些环境的。

# 环境配置
前提是你的电脑必须是 windows-64bit，否则下面的配置并不适合你！（现在应该没有开发还用 32bit 了吧）

## 下载所需的软件
1. [codeblocks-16.01mingw-setup.exe](http://www.codeblocks.org/downloads/26)，版本号无所谓，但是一定要带 mingw 版本的，因为这样就不用我们自己配置编译器了
2. [mingw64](https://sourceforge.net/projects/mingw-w64/)

## 安装
对于 codeblocks，一路 next 安装即可。
对于 mingw64，在安装时则需要特别注意，弹出安装界面后，会有一个设置界面：

| Settings      |        |
| ------------- | ------ |
| Version       | ...    |
| Architecture  | X86_64 |
| Thread        | posix  |
| Exception     | seh    |
| Build version | ...    |

注意这里的 Architecture 一定要选 X86_64，其它默认就好。

## 配置
打开 codeblocks，点击菜单中 settings->compiler，在这个窗口中，选择 toolchain excutables，在 compiler's installation directory 一栏选择前面 mingw64 的安装路径。下面的 program files 的选择依照图示即可。

![codeblocks 配置 compiler](http://img.blog.csdn.net/20180105175637924?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvWmFjaGF4eQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图示的配置是一种可行的方法，同时可以尝试如下配置：
| program files           |                               |
| ----------------------- | ----------------------------- |
| C compiler              | x86_64-w64-mingw32-gcc.exe    |
| C++ compiler            | x86_64-w64-mingw32-g++.exe    |
| Linker for dynamic libs | x86_64-w64-mingw32-g++.exe    |
| Linker for static libs  | x86_64-w64-mingw32-gcc-ar.exe |
| Debuger                 | default                       |
| Resource compile        | windres.exe                   |
| make programe           | mingw32-maker.exe             |

## 设置调试器
这一步并不是必须的，但是如果你写的 c 代码出了问题，那么就需要调试了。

打开 codeblocks，点击菜单中 settings->debugger，点击窗口左边的 gdb/cdb debugger，然后选择 create congfig，新建一个 mingw64 的 debugger ，这个 debugger 是我们自定义的，自己取个名字好了，同时将 debugger type 设置为 GDB，并指定该 debugger 的路径，将 executable path 设置到上面安装的 mingw64 的 bin 目录里的 gdb.exe 。点 OK 然后回到 compiler 设置菜单里，把 debugger 设置为新建的 gdb 。

至此，window 下 JNI 的开发环境就搭建成功了。