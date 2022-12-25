---
title: Maven简介
date: 2017-10-20 19:57:18
tags: 工具
---

最近在看 Gradle 相关的知识，看到介绍 Gradle 的开篇总是会拿来和 Maven 做对比，于是就好奇了解下 Maven 的使用，本篇文章只是对 Maven 很浅显的一个介绍，知道 Maven 是怎么回事，并不会对 Maven 做深入的探讨，因为我们的重点还是 Gradle。


Ant，Maven，Gradle 都是优秀的项目管理工具，做 Android 开发时应该都知道整个项目是用 gradle 进行管理的，平时用的最多的也就是往 gradle 配置文件中添加第三方库的依赖，仅此而已。接着我们直接点击了 run 的按钮等最终结果的输出了，而这些项目管理工具到底管理的是什么呢？我们不去看官方的话语，这里用大白话解释下，所谓项目管理，可以想象一个很大的项目，模块无穷多，这时就遇到问题了。
<!-- more -->
1. 项目中不可避免的要引入一些第三方的库
2. 每个模块之间也会产生一定的依赖关系
3. 每个模块都要进行独立的单元测试
4. 代码在其他人机器上不能跑，而在自己的机器上明明可以跑...
5. 项目需要在构建的过程而不是编码过程中加入一些自定义的功能
6. ......

这么多的问题，想想就头大，然而平时我们做的 hello world 根本不会遇到这些问题。如何解决？单靠程序员手动解决，简直累死，于是项目管理工具产生了，我们只需要在配置文件中进行简单的配置，即可完成以上复杂的流程。



# Maven 下载安装

略。。。

注意配置环境变量

配置好之后，用 `mvn -v` 查看是否安装成功并成功配置了环境变量。



# Maven 项目的默认目录结构

我们平时写的代码可能有自己的组织结构，要想将该项目改造为由 Maven 管理，那么就要按照 Maven 的管理规则来，否则 Maven 无法识别正确的路径，影响最终结果的生成。Maven 默认的文件结构如下

```
项目文件夹
	src
    	main
        	java
	test
		java
```

我们的平时写的代码放在 src/main/java/下，测试代码放在 test/java 下，当然这个结构也是可以通过 maven 来配置的，不过一般使用默认结构，而且现在大多数的 IDE 创建的工程也是按照这个结构组织的。



# pom.xml 简介

上面只是对代码的文件夹结构进行了约束，若要使用 maven 提供的 mvn 命令来管理该项目，还需要在项目文件夹下添加一个 pom.xml 的配置文件，其基本的结构如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.mymavenproj</groupId>
    <artifactId>mymavenproj-model</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>
</project>
```

这里有几个节点简单介绍下：

首先，project 的属性是默认的，所有的 pom 文件都一样，接下来是 modelVersion 节点，其值是 4.0.0，也是固定的。

- groupId：项目包名（一般为公司域名反写+项目名）
- artifactId：项目名-模块名（maven 是的一个优势就是对项目多模块之间的管理，不同的模块都可以用 maven 进行管理）
- version：项目的版本号

以上三个节点称之为一个组件的 **坐标**，因为我们可以通过这三个属性唯一的确定一个模块或者依赖。

- dependencies：用来进行依赖管理，具体见下一节



# 依赖管理

现在的项目中难免会用到一些第三方的依赖库，早期的做法是直接将 jar 包拷贝下来，放到工程目录的 libs 文件夹下，但是 jar 包一多，尤其是对于团队开发，将 jar 包们来回拷贝也是麻烦，而且还可能因为不同的小组使用了不同版本的 jar，导致项目最终无法通过编译，好在 maven 为我们解决了这个问题，不用将 jar 包拷贝来拷贝去，同时只要指定 jar 包的版本号，即可是团队中的 jar 包保持一致，不同的小组只需要同步这个 pom.xml 文件即可。

在 pom.xml 文件中，有一个 dependencies 的节点，在这个节点中可以将我们所有的依赖配置进去。每个依赖都由一个单独的 dependency 节点组成，该节点下有三个重要的属性：

- groupId：项目包名 
- artifactId：项目名-模块名 
- version：项目的版本号

这和上一节介绍配置我们自己项目时需要的三个属性是相同的，通过在三个属性，就能唯一确定一个依赖项。

eg：maven 的项目管理中包括测试这一项，这就需要我们在依赖中添加关于测试相关的框架

我们可以在 [maven center](http://search.maven.org/) 中寻找我们所需的项目，这是 Maven 为我们提供的仓库，有了它，我们就不必到 Junit 的官网下载 jar 包了，而是通过 maven 统一管理，在这个网页中搜索 junit，找到 http://search.maven.org/#artifactdetails%7Cjunit%7Cjunit%7C4.12%7Cjar 这里提供了如何向我们自己的项目中添加 junit 的依赖。

添加完 junit 的依赖后，我们的 pom.xml 的内容就是上一节中的样子。



# 常用命令

经过前面的配置，一个简单的 Maven 项目已经配置好了，那么接下来考虑我们要做什么？

- 代码要被编译为 class——compile
- 代码要进行测试——test
- 代码可能需要进行打包生成 jar 或者其它形式——package
- 代码上传到本地仓库——install
- 清除上次构建产生的结果——clean
- ......

为满足上述的要求，可以使用后面提供的命令 `mvn  xxx`即可。

构建所产生的结果放在工程目录下的 target 子文件夹中

这里特别要说明一下 install 的命令，假设我们开发的这个模块是日志记录模块 log，那么执行 package 命令之后就会生成对应 的 jar 包，接下来就要考虑如何将该 jar 包提供给其它模块来使用了，假设这里有一个网络模块 net，需要依赖 log 模块，那么需要在 net 的 pom.xml 的 dependency 中加入 log 对应的坐标即可。仅仅是加入这个依赖，那么 net 模块是如何找到 log 的代码呢？

这里分为两个步骤，首先 Maven 会从本机的本地 Maven 仓库去找，如果找到就使用，如果没找到，那么从 [maven center](http://search.maven.org/) 去找，如果还找不到，就直接报错了。这个 Maven 本地仓库的地址是：`C:\Users\UserName\.m2\repository`这是 windows 下的默认路径，当然也可以修改为其它路径。

那么这个 install 命令的作用就是将 package 生成的 jar 包拷贝到该路径下，这样在接下来 net 进行编译的时候就可以找到对应的 log 的依赖了。



# Maven 生命周期

了解了上面常用的命令，我们还需要知道 Maven 工程构建的生命周期，上述的命令执行其实是有先后顺序的，eg：`mvn install`如果执行该命令，其实会先依次执行： `compile -> test -> package -> install`

一般情况下，直接点击 install，等待打包即可，中间任何环节出现问题，都会停止构建，并输出对应的信息。



同时要注意的是，如果运行 package 命令，中间势必要经过 test 的流程，这就要求我们在当前项目中必须要依赖 junit，即使我们并没有写任何测试代码，否则 test 流程会失败。



# 在 idea 中使用 Maven

前面讲解的流程都是手动构建了项目目录，然后使用 maven 命令行对项目进行构建的，通过这种方式可以更加清楚的了解 maven 构建过程中的一些细节，但是这种方式已经很少手动去都建了，所有的大型项目都是通过 IDE 来写的，而且目前几乎所有的 IDE 创建的工程目录结构默认都是符合 Maven 规范的，所以我们也不用手动去创建符合 Mavne 标准的目录结构了。

接下来以常用的 IDE——idea 来讲解如何在项目中使用 Maven。

新建一个工程，类型为 empty，然后在该工程下创建三个 module，依次为 log，net，ui。module 的类型可以选择 java,也可以选为 maven。这两个类型的区别在于 maven module 会比 java module 多一个 pom.xml 文件，仅此而已，其它的都一样。我们既可以直接创建一个 maven 的 module，也可以选择创建好 java module 之后，在该 module 的根目录下手动添加一个 pom.xml，然后执行同步，其最终结果都是一样的。

但是如果你真的想让 maven 来管理项目，那么还是建议直接创建 Maven 项目，因为先创建 java module，然后在添加 pom.xml 文件时，识别比较慢，而且 java module 创建的目录下只到了项目目录/src，而 src 下面没有 main 和 test 的文件夹，还是需要我们手动创建

目前我们已经创建了 log，net，ui 三个 module，三个模块的 pom 文件如下：

log 模块：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-log</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>

</project>
```

net 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-net</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>

</project>
```

ui 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-ui</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>

</project>
```



# 依赖传递

目前我们已经创建了 log，net，ui 三个 module，然后让 net 依赖 log，ui 依赖 net。三个模块的 pom 文件如下：

log 模块：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-log</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>

</project>
```

net 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-net</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
        <dependency>
            <groupId>com.zx.testmaven</groupId>
            <artifactId>testmaven-log</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

ui 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-ui</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
        <dependency>
            <groupId>com.zx.testmaven</groupId>
            <artifactId>testmaven-net</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```

目前的依赖关系是：ui->net->log；

我们依次对 log，net，进行 install，对 ui 进行 package。此时可以发现 ui 的 dependencise 中依赖了 net，同时又依赖了 log。这就是所谓的依赖传递，虽然 ui 并没有直接依赖 log，但是其依赖的 net 中却依赖了 log，因此 ui 也就依赖了 log。这样的话，在 ui 中编写代码时也可以直接使用 log 中的类了。



# 依赖的排除

上一节介绍了依赖的传递，但是现在有个问题，ui 中并不想依赖 log，因为在某些情况下，我们在 ui 中自己写了一个类和 log 中的类是同名的，恰好方法也是同名的，如果没有注意，可能会引入 log 中的类，而我们的本意是依赖 ui 中自己写的类，这就在 import 时，带来不必要的麻烦，解决的方法是在 ui 的 pom 文件，对net 的依赖中，使用 exclusions 标签，将不想依赖的 log 移除，移除后的 pom 如下：

```xml
<dependency>
	<groupId>com.zx.testmaven</groupId>
  	<artifactId>testmaven-net</artifactId>
  	<version>1.0-SNAPSHOT</version>
  	<exclusions>
    	<exclusion>
      		<groupId>com.zx.testmaven</groupId>
      		<artifactId>testmaven-log</artifactId>
    	</exclusion>
  	</exclusions>
</dependency>
```





# 依赖冲突

产生依赖冲突的场景：

（1）A->B->C, A->C

A 依赖 B，B 依赖 C，而 A 有直接依赖 C，同时这两个 C 的版本还不一致，这就导致了依赖冲突。

解决该冲突的默认方法：

短路优先：默认解析路径短的，上例中，会采用 A 直接依赖的 C 的版本，而不是采用 B 中依赖的。

（2）A->B-X,A->C->X

A 依赖 B，B 依赖 X，同时 A 依赖 C，C 依赖 X，也会导致冲突

解决该冲突的默认方法：

路径相同，谁的 dependency 靠前就依赖谁，eg 在 A 的 dependency 中，先依赖的 B，那么间接引用的 X 采用的是 B 的。



# 依赖的范围

引入依赖后，该依赖起作用的范围也是不同的，其大体可以分为以下三种范围：

- 编译
- 测试
- 运行

这里所说的范围指的是

有的依赖仅仅是在测试的时候引入的，有的依赖仅在编译的时候使用，有的依赖仅在运行时使用，或者有的依赖是在其中的三种或者两种是有效的，为了避免依赖在其无用的范围内被错误的使用，maven 为我们提供了以下 6 种 dependency 节点中可选的 scope 属性，其对应的值分别是：

- compile：默认级别，对于编译，测试，运行三种 classpath 均有效；
- provided：仅在编译和测试时有效，在运行是不会被打入包中
- runtime：在测试和运行时有效，典型的应用就是 jdbc 驱动的实现；
- test：仅在测试范围有效；
- system：类型 provided，在编译和测试时有效，但是该特性与本地系统相关联，系统移植性差
- import：导入范围，仅适用于 dependencyManagement 标签下，表示从其它的 pom 中导入 dependency 配置

而我们也看到，在上述的三个模块中，均使用了 junit 的框架，而该框架只是在测试的范围内使用的，因此我们可以在 dependency 节点中引入 scope 节点，将其值设置为 test

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.10</version>
    <scope>test</scope>
</dependency>
```

# 聚合 

我们要得到 ui 的 jar 包，要分别对 log 和 net 进行 install，这就要求我们必须了解各个模块是如何相互依赖的，然后在手动的去 install。有没有一种方法，通过一个命令，直接得到最后 ui 的 jar，答案就是聚合。

依旧在该工程中创建一个 maven 的 module，这里主要要将 packaging 标签改为 pom，默认值为 jar，同时在 modules 标签中，将上面的 log，net，ui 的 module 都包含进来。示例如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-aggreation</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>pom</packaging>

    <modules>
        <module>../Log</module>
        <module>../Net</module>
        <module>../UI</module>
    </modules>
</project>
```

此时在依次执行该 module 的 celan->package 就会直接得到 ui 的 jar。

注意：使用 package，并没有将 log 和 net 的 jar 上传到本地 maven 仓库中，但却可以打出 ui 的包。



# 继承

在上面的三个模块中都是用了 junit，我们可以将其抽象出来，定义到一个父 pom 中，然后让这三个模块都继承这个父 pom，而自身的 pom 中不需要引入 junit 的依赖了。

定义一个 parent 的 maven 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-parent</artifactId>
    <version>1.0-SNAPSHOT</version>

    <packaging>pom</packaging>

    <properties>
        <junit.version>4.1.0</junit.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>${junit.version}</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>
```



注意点：

1. 其 packaging 的值为 pom
2. 是用 dependencyManagement 标签管理 dependencies，然后在其下的 dependency 中将 junit 填入
3. 可以定义为 junit 的版本定义一个属性，然后在其 version 中进行引用（可选）



接下来，要让三个模块来继承这个 parent，以 log 为例，进行如下修改：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zx.testmaven</groupId>
    <artifactId>testmaven-log</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>com.zx.testmaven</groupId>
        <artifactId>testmaven-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

<!--    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
        </dependency>
    </dependencies>-->

</project>
```

引入了 parent 标签，同时将依赖中的 junit 删除掉。net，ui 模块也是类似的处理。

接下来，先对 parent 模块进行 install，然后再用 aggreation 模块，进行 install，此时 ui 的 jar 成功的安装在本地 mave 仓库中。

