---
title: 易错面试题
date: "2019-05-12"
description: ""
url: /blog/go/
image: "/blog/go/interview_question.jpeg"
---
汇总部分易错Go语言面试题

<!--more-->
#### 1
以下代码的输出内容为

``` go
package main
import (
    "fmt"
)
func main() {
    defer_call()
}
func defer_call() {
    defer func() { fmt.Println("打印前") }()
    defer func() { fmt.Println("打印中") }()
    defer func() { fmt.Println("打印后") }()
    panic("触发异常")
}
```
答案

打印后
打印中
打印前
panic: 触发异常