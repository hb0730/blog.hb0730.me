---
title: Golang编译时使用·ldflags加入版本信息等
date: 2021-11-25 11:09:11
author: hb0730
authorLink: https://blog.hb0730.com
tags: [ 'Golang' ]
categories: [ 'Golang' ]

---

# 概述

在做 `drone`插件时，发现了一个库[drone-dingtalk-message](https://github.com/lddsb/drone-dingtalk-message) 在使用 [goreleaser](https://goreleaser.com/) 编译时只用了`ldflags`指令，于是了解了一下

# 编译时加入自定义信息到程序中

通常情况下，我们直接使用`go build`来编译程序，要想加入自定义信息到程序中，很简单，编译时使用`-ldflags`就行。

例如有如下程序test.go：

```golang
package main

import "fmt"

var AAA string
var BBB string

func main() {
    fmt.Println("AAA =", AAA)
    fmt.Println("BBB =", BBB)
}
```

在编译时加入-ldflags:

`go build -ldflags "-X 'main.AAA=111' -X 'main.BBB=222'"`

运行程序：

```console
./test

AAA = 111
BBB = 222
```

可以看到AAA和BBB两个变量的值变成了111和222，也就是在编译时加入的信息。

文档中可以看到相关参数的说明：

[hdr-Compile_packages_and_dependencies](https://pkg.go.dev/cmd/go#hdr-Compile_packages_and_dependencies)

[Command Line](https://pkg.go.dev/cmd/link)

```
-ldflags '[pattern=]arg list'
    arguments to pass on each go tool link invocation.

-X importpath.name=value
    Set the value of the string variable in importpath named name to value.
    This is only effective if the variable is declared in the source code either uninitialized
    or initialized to a constant string expression. -X will not work if the initializer makes
    a function call or refers to other variables.
    Note that before Go 1.5 this option took two separate arguments.
```

这里的`importpath.name=value`可以理解为`包名.变量名`，比如上面的`main.AAA`就表示`main包`下的`AAA变量`。

* [drone-plugin-notice](https://github.com/hb0730/drone-plugin-notice) drone 通知插件
