---
title: go语言快速上手
date: 2021-04-18 00:22:35
tags: 编程语言
---

> 非 go 语言深入教程,只是作一个笔记整理,方便在使用时,能快速的编写代码; 其中罗列了编程语言中最基本的:环境安装,变量,函数,if/else, for 循环, 集合类使用,以及go语言的基本特性

<!--more-->

# 导入包
两种方式
```go
// 第一种方式
import "xxx"
import "yyy"


// 第二种方式,推荐使用,同时可以使用多个括号,对导入进行分组,使代码结构更清晰
import {
    "xxx"
    "yyy"
}

import {
    "zzz"
    "mmm"
}
```

不可以有冗余 import,如果发现 import 的包在代码中没有使用,那么

# 访问权限
不同于 Java 有对应的 public,private 等关键字,go 中,只有包中的变量或方法是以大写字母开头的,才可以被外部引用

# 变量
- 使用 `var` 关键字定义变量, 后面是变量名称, 变量类型在最后面
- 初始化变量,可以不声明类型,go 可以根据初始化的值,推断变量类型
- 在函数中定义的变量,同时进行赋值,只是在函数中使用,有更简单的声明方式
## 定义
```
var i int
var str1, str2 string
```

## 初始化
```go
var i, j int = 1, 2
var c, python, java = true, false, "no!"
```

## 函数中的短变量声明
```
func main() {
	k := 3
	c, python, java := true, false, "no!"
	fmt.Println(j, k, c, python, java)
}
```

## 外部变量,分组声明
类似外部 import 导入一样,外部变量也可以分组声明
```
package main

import (
	"fmt"
	"math/cmplx"
)

var (
	ToBe   bool       = false
	MaxInt uint64     = 1<<64 - 1
	z      complex128 = cmplx.Sqrt(-5 + 12i)
)

func main() {
	fmt.Printf("Type: %T Value: %v\n", ToBe, ToBe)
	fmt.Printf("Type: %T Value: %v\n", MaxInt, MaxInt)
	fmt.Printf("Type: %T Value: %v\n", z, z)
}
```

## 基本类型变量的默认值
string 类型的默认值是 "" ,而不是 nil

## 类型转换
需要显式声明,加强了编译时检测
```go
var i int = 42  // 定义变量
var f float64 = float64(i) // 进行类型转黄, 变量定义钱的 float64 不可忽略
// 或者
var f:= float64(i)

```

# 方法定义
- `func` 关键字定义方法,不同于 kotlin 中的 `fun` ... 
- 返回值类型在括号后面...
- 函数中的参数,如果出现连续两个及以上,类型相同,则可以只写最后一个参数的类型
- 返回值可以有多个,用()包裹
- 在函数定义时,可以定义返回值变量,其在函数内可以直接使用, 最后以 return 语句结,较大的函数中不推荐使用,可读性差
```go
package main

import "fmt"

func add(x int, y int) int {
// func add(x, y int) int {  // 等价的
	return x + y
}

// 返回值可以有多个
func swap(x, y string) (string, string) {
	return y, x
}


// 在函数定义时,定义返回值变量, 其在函数内可以直接使用, 最后以 return 语句结束
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return  // return 语句不能省略
}

func main() {
	fmt.Println(add(42, 13))
}
```