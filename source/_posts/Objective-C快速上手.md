---
title: Objective-C快速上手
date: 2017-06-05 15:28:42
tags: 编程语言
---

> 本文是 Objective-C 的快速上手文章，只要你之前有 Java 或者 C++ 的基础，相信入门 Objective-C ，只需要很短的时间。

# OC 语法概览

## 源代码文件拓展名对比

|         | 头文件  | 实现文件 |
| ------- | ---- | ---- |
| c       | .h   | .c   |
| c++     | .h   | .cpp |
| oc      | .h   | .m   |
| oc&&c++ | .h   | .mm  |



## 类的定义

```objective-c
@interface MyClass:NSObject

@end
```

定义一个类，以@interface MyClass:NSObject 开头，以@end 结尾，类名就是 MyClass，后面跟一个冒号，其含义就是 Java 中的 extends 关键字， 表示继承自 NSObject 类



遇到@符号，编译器会进行相应的预处理。

声明之中只有属性的定义，方法的声明，并没有方法的具体实现，具体实现是在类的实现中。从这个角度看，类的声明又很像是 Java 中的接口。类的声明一般是放在头文件中`.h`中

## 属性的定义

```objective-c
@interface Person:NSObject
	@property NSString *firstName;
	@property NSNumber *age;
	@property int id;
	@property (readonly)NSString * lastName;
@end
```

定义属性必须以@property 开头，然后是属性的类型，后面是变量名，如果变量是以`*`开头的，那么表明这个变量是一个指针，指向的是一块对内存中的区域；

同时，属性也有基本类型，eg：int，double 等，这表明的是一个基本类型；

用(readonly)前缀的变量，表明该变量时只读的，这和 Java 中的 final 关键字很类似



## 方法的定义

减号方法（普通方法，又称为对象方法）

加号方法（类方法，又称为静态方法）



## 类的实现

类的声明和实现是分开的，也就是说声明之中只有属性的定义，方法的声明，并没有方法的具体实现，具体实现是在类的实现中。从这个角度看，类的声明又很像是 Java 中的接口



## 完整的例子

定义类 `Person.h`

```objective-c
@interface Person:NSObject
-(void)sayHello;
@end
```



实现类`Person.m`

```objective-c
# import "Person.h"

@implementation Person
-(void)sayHello{
 	NSLog(@"hello,world");
} 
@end
```

在实现方法中，有一个 NSLog 的方法，这个方法是一个静态方法，类似于 Java 中的 System.out.println 方法；

在"hello world"字符串之前，加一个@符号，表明这是一个 OC 类型的字符串，OC 是完美支持 C 语言的，如果不加@符号，那么表明这是一个 C 语言类型的字符串（其实就是一个字符数组）。



## 简单的 main 方法

```objective-c
#import <Foundation/Foundation.h>

int main(int argc, char const *argv[])
{
	/* code */
	@autoreleasepool{
		NSLog(@"hello world");
	}
	return 0;
}
```



# 变量

## 基本数据类型

- int：4byte
- float：4byte
- double：8byte
- char；1byte
- boolean ：和其它语言不一样的是，oc 中 true 是用 1 来表示的，false 是用

限定词：

- long：long int a，表示加长版的 int，简写为：long a
- long long：long long int a，表示加长版的 long，简写为：long long  a
- short：short int a，表示简短版的 short，简写为： short a
- unsigned：unsigned int a，表示无符号整形
- signed：signed int a；表示有符号的整形

字符串:并不是基础类型，但是可以平时使用很多

OC 字符串类型：NSString:@"hello"，在打印时使用%@

C 语言字符串类型："hello"，在打印时，使用%s



# 条件控制

## if

关于 OC 中的布尔值，和其它语言不同，并不是 true、false，而是用 0 来表示 false，如果 if 中的条件表达式中的值不是 0，其它所有的数字，或者字符串都可以表示 true；

## goto

和 C 语言的 goto 语法相同，定义一个标签，只要不是关键字都可以，后面的代码用花括号包起来，不包也可以；然后在想要跳转的地方进行 goto label 的跳转；

标签并不一定要定义在 goto 语句之前出现，后面也是可以的；



# 循环语句

for、while 和 C 语言完全一样



# 函数

定义：同 C 语言

调用：同 C 语言



# Objective-C 面向对象

OOP：面向对象编程

OOA：面向对象分析

OOD：面向对象设计



 ## 创建类和对象

在 Xcode 中创建一个类，右键 new File，选择 iOS 中的 Source，选择 Cocoa Touch Class,选择继承自 NSObject，这样的话就会生成两个文件，分别是一个 .h  和 .m 文件，h 文件中写类的声明，m 中写类的实现。

```objective-c
// Person.h
# import<Foundation/Foundation.h>

@interface Person : NSObject{
	NSString *name
	int age;
}

//定义一个属性，这个属性就相当于对成员变量的 get/set
@property(nonatomic,strong)NSString *personName;

//上面属性的定义的本质就是定义了如下方法：
/*- (void) setName:(NSString *)name;
- (NSString *)getName;*/
@end
```



```objective-c
// Person.m
#import "Person.h"

@implementation Person

@synthesize personName = name;

- (instancetype)init{
	self = [super init];
	if (self)
	{
		name = @"张三"；
	}
	return self；
}

/*- (void) setName:(NSString *)name{
	personName = name;
}

- (NSString *)getName{
	return personName;
}*/

@end
```



在 main 方法中使用该对象；

```objective-c
#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, char const *argv[])
{
 	@autoreleasepool{
		Person *p1 = [[Person alloc]init]; 
        p1.personName = @"张三";
      	Person *p2 = [Person new];
 	}
	return 0;
}

```



解析`Person *p1 = [[Person alloc] init];  `

首先，Person 是类名，后面跟变量名，但前边必须有一个 `*`,表示指针，所以在 OC 中，所有的对象都是一个指针，这其实和 Java 是类似的，因为 Java 中的对象的引用，实际上就是指向对内存中的地址，其实也就是一个指针。

- 对于静态方法：[类名 方法名]
- 对于非静态方法：[对象名 方法名]

alloc 是一个方法名，用来分配内存，init 也是一个方法名，用来初始化对象；

另一种初始化对象的方法是 [类名 new]，直接使用一个 new 的关键字，就代表了 alloc 和 init 两个方法，直接申请内存并初始话对象，但是这种方法与整体编码风格不搭，所以还是使用先 alloc 再 init 的方法。



 ## 成员变量和属性

类内使用成员变量，类外使用属性

因为成员变量是只能在类内使用的，属性存在的意义是让类外访问成员变量，充当成员变量的外部访问接口。从这个角度来分析，这又有点像 Java 中的 get/set 方法；

所以我们在 h 文件中定义了一个属性，名字叫做 personName，那么我们直接在类外对这个属性赋值或者取值就可以了，如果没有定义属性，那么我们就必须自己手写 get/set 方法，代码量增加了。然而如何将属性和成员变量进行关联呢？

成员变量名：name

属性名：personName

为了建立关联，我们要在 m 文件中定义关联：`@synthesize personName = name;`

如果成员变量名和属性名一样的话，eg 都是 name，那么关联更为简单：

`@synthesize name;`

但是这样写的话，带来一个困扰，就是在类中调用的究竟是成员变量还是属性，这是两个对象，这给代码的编写带来迷惑，所以不建议将属性名和成员变量名写成一样的。

在类内是完全没有必要调用属性的，因为成员变量在类内是完全可用的，所以我们在类内应该使用成员变量，属性是给类外使用的。

苹果官方推荐的做法是，成员变量使用 _name，而属性使用 name，以作区分，当然这只是一个建议，并不一定必须要这样；然后在 m 文件中使用 @synthesize 关键字进行对应，这是较老的版本的做法。

在新版本中则不需要这么做，如果我们想要将一个属性和成员变量进行对接，直接定义一个属性即可，也不需要再 m 文件中使用 @synthesize 关键字进行对接，SDK 已经自动帮我们生成了`_ 属性名`的成员变量，在类中我们只可以使用 _name 来获取成员变量。

因为成员变量是在类内才能进行调用的，所以现在成员变量在 m 文件中定义和在 h 文件中定义是没有区别的。建议在 m 文件定义吧，但是属性是一定要在 h 文件中定义的。



## 方法

### 方法的声明

方法的定义是在 h 文件中，这里只有一个方法的声明，声明的形式：

方法类型 (方法返回值类型) 方法名 : (参数 1 类型) 参数 1 名字  

```
-(returnType)methodName:(typeName) variable1 :(typeName)variable2;
```

- 方法类型，可选的有 + 或者 -，其中前者代表类方法，后者代表对象方法
- 方法返回值类型：要用括号包裹起来，eg：(void) ，(int) ，(NSString *)等
- 方法名：和其它语言相同
- 方法参数：没有参数，那么方法名后面就不用跟冒号了，如果有参数，那么方法名后边要跟一个冒号，然后



eg：

```
- void showName;
- void showName : (NSString *) name;
- void showName : (NSString *) name andAge:(int) age;
```

### 方法的实现

方法的实现是在 m 文件中；





### 方法的调用

使用[ ] ，完成调用，分为两种情况：

- 对于静态方法：[类名 方法名]
- 对于非静态方法：[对象名 方法名]

```objective-c
Person p1 = [[Person alloc] init];
[p1 showName]；
[p1 showName: @"zx"]；
[p1 showName: @"zx" andAge: 24]；
```

### 关于 init 方法

```objective-c

# import<Foundation/Foundation.h>

@interface Person : NSObject{
 
}
- (id) init;
- (instancetype)init;
- (instancetype)initPerson:(NSString *)name :(int)age;
@end
```
关于 init 方法，我们可以在类的声明中重写，其返回值类型可以是 id，表示任何类型；返回值类型可以使 instancetype，表示当前类型；单纯的初始化方法的话，其实都可以，但是要赋值的时候可能会出错，所以推荐使用后者；因为我们所有的类其实都是 NSObject 的子类，所以 init 方法不在类声明中也是可以的，如果声明的话，就相当于重写了；这里重写 init 方法的意义在于给初始化方法提供多个参数，供具体的类来使用。例如上面的 initPerson，具体实现略；

那么在使用时就可以用下面的方法来调用：

```objective-c
...
Person * p1 = [[Person alloc]initPerson:@"zx":20];
...
```


## 封装

屏蔽内部实现细节，只提供一个使用的接口。和 Java 无异；

### 访问修饰符

可用来修饰成员变量（注意不是属性）

- @public :在类内，类外都可以使用，并可以被继承；在类外的使用方法是：p1->name
- @protected：默认访问修饰符，在类内可以使用，类外不可被使用，可以被继承
- @private：只能在类内使用，不可以被继承；
- @package：框架权限，在框架内相当于 protected，在框架外，相当于 private；



**注：方法是没有访问修饰符**

这个用法完全和 C 语言是一样的，如果不想让人在外面使用，可以不在 h 文件中申明该方法，那么在类外就无法调用该方法了（类外引用的永远都是 h 文件，而看不到具体实现），调用直接报错；而我们可以在 m 文件中写一个方法的声明，这样的话就可以在 m 文件内部被其它方法调用了。



## 继承

OC 中也是单继承的，要想实现多继承的效果，可以使用协议；

关于继承，我们在之前其实已经见到过了，就是继承了 NSObject；NSObject 是所有类型的基类；



继承的语法：

1. 在 h 文件中

   这时 import 的是父类的 h 文件，并且在 @interface 后面类名，后面加冒号，然后跟父类名字；

2. 在 m 文件中，我们可以对父类中在 h 文件中声明的方法进行一个覆盖，不覆盖默认使用的是父类中的方法



父类在 h 文件中声明的方法才可以被子类继承，如果没有在 h 文件中声明，那么表示只有父类自己内部可以用，外部，子类均不可用。



## 多态

父类引用指向子类对象；

OC 中不支持方法重载，只能重写

其它特点和 Java 无异。
