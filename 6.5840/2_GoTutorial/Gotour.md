---
layout: default
title: GoTour
description: GoTour一些知识点记载
---

https://tour.go-zh.org/这个链接是中文版Go指南的官网，下面只记录一些比较抽象的知识点，不会全部记载。

# 1. Methods，Interface

## 1.1 Error

Go 程序使用 `error` 值来表示错误状态。下面给出一个具体例子：

```go
package main
import ("fmt" "time")
type MyError struct {
	When time.Time
	What string
}
func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s",
		e.When, e.What)
}
func run() error {
	return &MyError{ time.Now(),"it didn't work",}
}
func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```

在Go语言中，`error`是一个内建的接口类型，它定义了一个方法`Error() string`。任何实现了这个方法的类型都可以被认为是一个`error`类型。在您的代码中，`MyError`类型实现了`Error()`方法，所以它可以被当作`error`类型来使用。

当您在`run()`函数中返回一个`*MyError`类型的值时，由于它实现了`error`接口，所以可以被当作`error`类型来使用。这就是为什么`run()`函数的返回值类型可以是`error`，尽管我们实际上返回的是一个`MyError`类型的值。

至于为什么调用`fmt.Println(err)`就相当于调用了接收者函数`Error()`，这是因为当我们尝试打印一个错误时，Go语言会自动调用该错误类型实现的`Error()`方法来获取错误的字符串表示形式。所以，在您的代码中，当我们尝试打印错误时，Go语言会自动调用您定义的`MyError.Error()`方法来获取错误信息。这就是为什么打印错误时会输出您在`Error()`方法中定义的字符串。这种机制使得我们可以自定义错误信息的格式和内容。

## 1.2 Error练习

从[之前的练习](https://tour.go-zh.org/flowcontrol/8)中复制 `Sqrt` 函数，修改它使其返回 `error` 值。

`Sqrt` 接受到一个负数时，应当返回一个非 nil 的错误值。复数同样也不被支持。

创建一个新的类型

```
type ErrNegativeSqrt float64
```

并为其实现

```
func (e ErrNegativeSqrt) Error() string
```

方法使其拥有 `error` 值，通过 `ErrNegativeSqrt(-2).Error()` 调用该方法应返回 `"cannot Sqrt negative number: -2"`。

**注意:** 在 `Error` 方法内调用 `fmt.Sprint(e)` 会让程序陷入死循环。可以通过先转换 `e` 来避免这个问题：`fmt.Sprint(float64(e))`。这是为什么呢？

修改 `Sqrt` 函数，使其接受一个负数时，返回 `ErrNegativeSqrt` 值。

```go
package main

import ("fmt""math")

type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintf("cannot Sqrt negative number: %v",float64(e))
}

func Sqrt(x float64) (float64, error){
	if (x < 0) {
		return 0,ErrNegativeSqrt(x)
	}
	z := 1.4 
    prez := 0.0
	for z - prez > math.Pow(2,-20) {
		prez = z
		z -= (z*z - x) / (2*z)
		fmt.Println("current z is:", z)
	}
	return z,nil
}

func main() {
	fmt.Println(Sqrt(2))
	_,err := fmt.Println(Sqrt(-2))
	if (err != nil) {
		fmt.Println(err)
	}
}
```

[back](../../index.html).