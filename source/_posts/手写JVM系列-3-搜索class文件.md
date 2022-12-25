---
title: 手写JVM系列(3)-搜索class文件
date: 2017-05-08 15:01:12
tags: JVM
---

> 本节的目的是根据`-cp`命令指定的class路径以及后面的`ClassName`,找到对应的class文件,然后简单的打印出该 class 文件的字节码.完整源码[classpath包中](https://github.com/zachaxy/JVM)

<!--more-->

# 关于classpath

`Java`虚拟机规范并没有规定虚拟机应该从哪里寻找类，因此不同的虚拟机实现可以采用不同的方法。Oracle的Java虚拟机实现根据类路径（class path）来搜索类。按照搜索的先后顺序,类路径分为以下三部分:

- 启动类路径(bootstrap classpath):默认对应jre\lib目录，Java标准库(大部分在rt.jar里)位于该路径
- 扩展类路径(extension classpath):默认对应jre\lib\ext目录，使用Java扩展机制的类位于这个路径
- 用户类路径(user classpath):自己实现的类，以及第三方类库则位于用户类路径

> 这里特别说一下:用户类路径的**默认值**就是当前目录,这也是平时我们配置Java开发环境时,网上很多教程会说要添加一个`classpath`变量,并将其变量值设置为 `.` (表示当前路径),其实这个根本没有必要,因为虚拟机默认是在当前路径下寻找class文件的.所以在刚开始学习Java编程时,使用文本编辑器写代码,只需要配置一个`JAVA_HOME`即可,如果是用`IDE`,那么恐怕连`JAVA_HOME`都不需要,只要你安装了`JDK`,在`IDE`中配置其路径即可,至于`classpath`,都是由`IDE`的脚本自动写好的,不同的`IDE`有自己的路径,我们只需要在`IDE`中写好代码,点击运行即可.



我们这里编写的`JVM`是脱离`IDE`的,所以不能指望`IDE`来帮我们寻找路径,经过上面的分析,一个想法就是配置一个`classpath`的环境变量,将所有的`class`文件都丢到这个路径下,但是这样做并不灵活,更好的办法是给`java`命令传递`-classpath`（或简写为-cp）选项。`-classpath/-cp`选项的优先级更高，可以覆盖`CLASSPATH环境变量`。



## java命令中的classpath 选项

 `-classpath/-cp`选项既可以指定目录，也可以指定`JAR`文件或者`ZIP`文件



## 添加指向JDK启动类路径

`Java`虚拟机将使用JDK的启动类路径来寻找和加载`Java`标准库中的类，因此需要某种方式指定`jre`目录的位置。命令行选项是个不错的选择，所以增加一个非标准选`-Xjre`选项。但是这一选项并不是必须的，目前已经实现了读取`JAVA_HOME`来寻找启动类路径的功能，所以`-Xjre`选项是在没有配置`JAVA_HOME`环境变量的情况下，使用的。如果本地机器上有该环境变量，则无需指定`-Xjre`



注意这个选项的效果类似于`-classpath`,只不过这个选项是我们为了实现`JVM`自己添加的选项,并不是`java`自带的,其功能是指向**`JDK`启动类路径**,而`-cp`是指定**用户类路径**.



但是不管怎样,最终的结果都是根据其执行的路径和后面所跟的类名,找到对应的`class`文件



# 代码编写

> 搜索class文件所实现的功能源码都在项目的[classpath包](https://github.com/zachaxy/JVM/tree/master/Java/src/classpath)下.



##  使用场景

首先考虑一下在命令行使用时的可能的场景:

1. 直接指定路径,后面跟类名:`java -cp aaa/bbb/ccc  ddd  arg1  arg2`
2. 指定类所在的jar文件的路径,后面跟类名:`java -cp aaa/bbb/ccc.jar  ddd arg1  arg2`
3. 指定一个模糊路径,后面跟类名:`java -cp aaa/bbb/*  ddd  arg1  arg2`
4. 指定若干个路径,后面跟类名(指定的类存在指定的某一条路径中,如果都存在那么以第一条为准):`java -cp aaa1/bbb/ccc;aaa2/bbb/ccc;aaa3/bbb/ccc;  ddd  arg1  arg2`



##  抽象类 Entry

```java
public abstract class Entry {
    //路径分隔符,在window下,使用 ; 分割开的  在Unix/Linux下使用: 分割开的
    public static final String pathListSeparator = System.getProperty("os.name").contains("Windows") ? ";" : ":";

    /**
     * 负责寻找和加载class文件
     *
     * @param className class文件的相对路径，路径之间用斜线 / 分隔，文件名有.class后缀
     */
    abstract byte[] readClass(String className) throws IOException;

    /**
     * @return 返回className的字符串表示形式;
     */
    abstract String printClassName();

    /**
     * 工厂方法,根据传入的path的形式不同,
     * @param path 命令行得到的路径字符串
     * @return 创建具体的Entry
     */
    static Entry createEntry(String path) {
        if (path != null) {
            if (path.contains(pathListSeparator)) {
                return new CompositeEntry(path, pathListSeparator);
            } else if (path.contains("*")) {
                return new WildcardEntry("");
            } else if (path.contains(".jar") || path.contains(".JAR") || path.contains(".zip") || path.contains("" +
                    ".ZIP")) {

                return new ZipJarEntry(path);
            }
            return new DirEntry(path);
        }
        return null;
    }

}
```



## 具体实现类

- DirEntry
- ZipJarEntry
- WildcardEntry
- CompositeEntry

这四个类是`Entry`的具体实现类,分别对应使用场景中的四种情况,下面会分析四种情况所要解决的问题,这里只贴出`DirEntry`的代码,先于篇幅原因,其它三种情况的具体实现,请查看本项目[源码](https://github.com/zachaxy/JVM)

###  DirEntry

这应该是最简单的一种使用场景了,构造方法中传入的字符串表示目录形式的类路径,这里拿到的直接就是指定的路径,那么先判断该路径是否存在,如果存在,那么和`className`拼接起来,使用`IO`流读取其中的字节码,并返回.

```java
public class DirEntry extends Entry {
    String absDir;

    public DirEntry(String path) {
        File dir = new File(path);
        if (dir.exists()) {
            absDir = dir.getAbsolutePath();
        }
    }

    @Override
    byte[] readClass(String className) throws IOException {
        File file = new File(absDir, className);
        byte[] temp = new byte[1024];
        BufferedInputStream in = null;
        ByteArrayOutputStream out = null;

        in = new BufferedInputStream(new FileInputStream(file));
        out = new ByteArrayOutputStream(1024);
        int size = 0;
        while ((size = in.read(temp)) != -1) {
            out.write(temp, 0, size);
        }
        if (in != null) {
            in.close();
        }
        if (out != null) {
            out.close();
        }
        return out.toByteArray();
    }


    @Override
    String printClassName() {
        return absDir;
    }
}
```



### ZipJarEntry

这种使用场景是指定了一个`zip文件`或者`jar包`的情况,那么需要解决的问题是如何拿到压缩文件中的文件,这里主要使用了`java.util.zip`包下的类来拿到文件,使用zipFile来读取文件和读取普通的文件夹类似,这里只管把zip文件当成文件夹就好

这里有一点要注意的是:

如果是`zip`文件,在获取`ZipEntry`的时候`ZipEntry ze = zipFile.getEntry(zipName + "/" + className)`

如果是`jar`包,在获取`ZipEntry`的时候,`ZipEntry ze = zipFile.getEntry(className)`



###  WildcardEntry

这种使用场景是指定了一个形如`aa/bb/*`的路径,这种路径表明我们的`class`文件在`aa/bb/`路径下的`jar`包中,所以我们只要遍历该路径下的所有以`.jar`结尾的文件,然后调用`ZipJarEntry`的实现方法,即可以获得字节码.



### CompositeEntry

这种使用场景是包含多个路径的情况,(eg:a1/b1/c1;a2/b2/c2;a3/b3/c3),那么遇到这种情况,需要将字符串分割成不同的子串,注意分割符在不同的系统下是不同的,这里仅仅实现`windows`,其它情况下视为`unix`

```java
public static final String pathListSeparator = System.getProperty("os.name").contains("Windows") ? ";" : ":";
```



当然分成的子串分别对应一个`DirEntry`,再调用`DirEntry`的方法来获取字节码.



## classpath对外的统一使用接口

上面我们已经实现了对于 classpath 路径的解析,这里在进行一步封装,提供一个对外的统一接口.

```java
public class ClassPath {
    //分别存放三种类路径
    Entry bootClasspath;
    Entry extClasspath;
    Entry userClasspath;

    //parse()函数使用 -Xjre 选项解析启动类路径和扩展类路径
    // 使用-classpath/-cp选项解析用户类路径

    public ClassPath(String jreOption, String cpOption) {
        parseBootAndExtClasspath(jreOption);
        parseUserClasspath(cpOption);
    }


    //这里参数传进来的是: C:\Program Files\Java\jdk1.8.0_20\jre
    void parseBootAndExtClasspath(String jreOption) {
        String jreDir = getJreDir(jreOption);

        //可能出现的情况是: jre/lib/*
        String jreLibPath = jreDir + File.separator + "lib" + File.separator + "*";
        bootClasspath = new WildcardEntry(jreLibPath);

        //可能出现的情况是: jre/lib/ext/*
        String jreExtPath = jreDir + File.separator + "lib" + File.separator + "ext" + File.separator + "*";
        extClasspath = new WildcardEntry(jreExtPath);
    }

    String getJreDir(String jreOption) {
        File jreFile;
        if (jreOption != null && jreOption != "") {
            jreFile = new File(jreOption);
            if (jreFile.exists()) {
                return jreOption;
            }
        }

        jreFile = new File("jre");
        if (jreFile.exists()) {
            return jreFile.getAbsolutePath();
        }

        String java_home = System.getenv("JAVA_HOME");
        if (java_home != null) {
            return java_home + File.separator + "jre";
        }

        throw new RuntimeException("Can not find jre folder!");
    }

    void parseUserClasspath(String cpOption) {
        userClasspath = Entry.createEntry(cpOption);
    }

    public byte[] readClass(String className) {
        className = className + ".class";
        byte[] data;
        try {
            data = bootClasspath.readClass(className);
            if (data != null) {
                return data;
            }

            data = extClasspath.readClass(className);
            if (data != null) {
                return data;
            }

            data = userClasspath.readClass(className);
            if (data != null) {
                return data;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        throw new RuntimeException("can't find class!");
    }

    @Override
    public String toString() {
        return userClasspath.printClassName();
    }
}
```

构造方法:传入解析到的 `-Xjre`来的路径和`-cp`的路径,当然在`Cmd`类中,这两个路径的初始值都是空字符串,而不是`null`,然后根据二者的值进行解析

之前也说过寻找 class 文件是由优先级的,依次是:`bootClasspath`-> `extClasspath` ->  `userClasspath`那么我们怎么实现这个加载顺序呢? 



首先:在`Cmd`类中,这两个路径的初始值是:

```java
String XjreOption = "";
String cpOption = "";
```



这里优先使用用户输入的 -Xjre 选项作为 jre 目录。如果没有输入该选项，则在当前目录下寻找 jre 目录。如果找不到，尝试使用 JAVA_HOME 环境变量。最终返回`bootClasspath`和`extClasspath`.

然后再根据 `-cp`的值得到一个`userClasspath`,然后再根据这三个`Entry`的顺序来加载类文件,一旦加载到,直接返回。



最终,使用`readClass`()方法就可以返回我们要加载的类文件字节数组。