---
title: mac下Android Studio快速上手
date: 2017-12-07 16:05:15
tags: 工具
---

# 常用快捷键
| 功能                      | 快捷键                 |
| ----------------------- | ------------------- |
| 深度搜索                    | shift + shift       |
| 查找类                     | cmd + o             |
| 查找当前方法                  | cmd + fn + f12      |
| 查找当前单词                  | cmd + f             |
| 全局搜索                    | cmd + shift + f     |
|                         |                     |
| 格式化代码                   | cmd + opt + l       |
|                         |                     |
| 快速插入下一行                 | shift + enter       |
| 快速补全行末分号                | cmd + shift + enter |
| 基础代码补全                  | ctl + space         |
| 万能键(导入包，自动修改等)          | alt + enter         |
| 删除不用的 import 包          | ctl + opt + o       |
|                         |                     |
| 快速移动一行                  | cmd + shift + 上下方向键 |
| 选中单词                    | opt + 方向键上          |
| 列编辑                     | opt + 鼠标选择          |
| 快速移动到错误处                | fn + f2             |
| 快速回到上一个浏览界面             | cmd + opt + <-/->   |
|                         |                     |
| 重写方法                    | ctl + o             |
| 创建构造方法                  | cmd + n             |
| 显示当前类的层次结构              | crl + h             |
|                         |                     |
| 翻译(需要安装ECTranslation插件) | ctl + l             |





[win 和 mac 在 idea 中快捷键的对比](http://blog.csdn.net/qq_35246620/article/details/53992312)



# 书签功能

每当我们查看工程较大的源码时,难免会在数十个个文件中跳来跳去,一会就跳晕了,好在 Android Studio 提供了书签功能,相信熟练了该用法后,肯定就离不开了.

两种书签：

- 匿名书签：无限个
- 具名书签：署签名为0~9或者A~Z间的一个字符作为助记符，因此数量有限



## win 环境

**添加书签**: 鼠标所在行，按 `F11`，则添加一个匿名书签，在按`F11`，取消；如果想添加具名书签，则在光标所在行按` ctrl + F11`，然后会弹出`0~9`或者`A~Z`的**助记符**选项，点击即可

**显示所有书签**:`shift + F11`

**书签之间的切换**:

如果是匿名书签，只能点 `shift + F11` 查看所有书签，然后选择

如果是具名书签，那么只需要 `ctrl + 助记符`则直接跳转到对应书签



## mac 环境

**添加书签**：鼠标所在行，按 `Fn + F3`，则添加一个匿名书签，再按`Fn + F3`，取消；如果想添加具名书签，则在光标所在行按 `opt + Fn + F3`，然后会弹出`0~9`或者`A~Z`的**助记符**选项，点击即可

**显示所有书签**:`cmd + Fn + F3`

**书签之间的切换**:

如果是匿名书签，只能点 `cmd + Fn + F3` 查看所有书签，然后选择

如果是具名书签，那么只需要 `ctrl + 助记符`则直接跳转到对应书签



# 新建文件时的注释模板配置

`File–>Settings–>Editor–>File and code Template`

选择右侧的 include 标签 ,打开 File Header , 按照提示添加对应的注释.

上面只是为新建的 Java 文件添加头部的注释,关于作者,创建时间, copyright 等,还有一种使用场景是新添加方法时的注释.可以参考[Android Studio 配置系列（一）：自定义代码注释](http://www.jianshu.com/p/a85062562cd7)





# 常用的几个插件

- Alibaba Java Coding Guidelines:代码规范检查
- CodeGlance: 右侧显示代码大纲
- ECTranslation: 取词翻译,安装后建议在快捷键设置中搜索 translation, 然后修改快捷键,mac 中使用的是 `ctr + l`



# 关闭 instance run

貌似目前问题还比较多,在  `preference -> Build,Execution -> instance run` 下关闭即可.



# mac 连接真机调试

1. 配置 adb 环境变量，找到 Android SDK 的位置，一般在：`~/Library/Android/sdk/`
  将下面两个路径添加到当前 shell 的配置文件的末尾，因为我将当前的 shell 换成了 zsh，因此我的配置文件时 ~/.zshrc

```
export PATH=${PATH}:~/Library/Android/sdk/platform-tools
export PATH=${PATH}:~/Library/Android/sdk/tools
```

记得 source ~/.zshrc
然后通过 adb version 命名查看是否配置成功！

2. 将手机连接到 mac 上，在终端输入 `system_profiler SPUSBDataType`，查看手机的信息；
  对应的输出可能有很多，我们只需要找包含 `Serial Number` 的那一组数据，找到该组数据的 `Product ID`对应的16进制数。
  然后打开 `~/.android/adb_usb.ini`，如果没有，则先创建该文件，并将`Product ID`对应的16进制数据写入该文件即可。


3. 重启 adb
  `adbkill-server`

4. 重新连接手机即可使用

5. 经过以上配置，以后如果还想调试其它手机，那么只需要将该手机的 `Product ID` 写入到 adb_usb.ini 文件中即可。

# adb server无法启动

在使用`adb shell`命令时，可能会遇到端口占用，无法启动adb server的问题。adb server使用的端口是5037，所以这时候可以列出使用5.07端口的进程，，然后杀掉占用5037端口的进程，再重新启动adb server

1. `netstat -ano | findstr "5037"  `,eg:找到的端口号是xxxx
2. `TASKLIST | findstr "xxxx"`
3. `adb shell`

目前已知会占用5037的服务有：金山毒霸，酷狗音乐，qq音乐等，这些软件有一个共同的特点：手机连接上电脑时，这些软件都会监听到。

