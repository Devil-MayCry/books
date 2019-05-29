---
title: Go语言中的反射
date: "2019-05-29"
description: ""
url: /blog/go/reflect/
image: "/blog/go/reflect/title.jpeg"
---
在计算机科学领域，反射是指一类应用，它们能够自描述和自控制。Golang语言实现了反射，官方自带的reflect包就是反射相关的。

<!--more-->

## interface 和 反射
在讲反射之前，先来看看Golang关于类型设计的一些原则

* 变量包括（type, value）两部分