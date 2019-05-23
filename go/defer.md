---
title: 深入理解defer
date: "2019-05-23"
description: ""
url: /blog/go/
image: "/blog/go/defer.jpeg"
---
defer语句在Go语言中用于延迟调用，但defer也可能引用主函数用于返回的变量，也就是说延迟函数可能会影响主函数的一些行为，这些场景下，如果不了解defer的规则很容易出错。

<!--more-->
## 引言
先来看几道题，可以先在心里默默地记住你的答案
``` go
//先来看看几个例子。
//例1：
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
 
//例2：
func f() (r int) {
    t := 5
    defer func() {
        t = t + 5
    }()
    return t
}
 
//例3：
func f() (r int) {
    defer func(r int) {
        r = r + 5
    }(r)
    return 1
}
``` 
想好了么？现在可以告诉你：例1的正确答案不是0，例2的正确答案不是10，如果例3的正确答案不是6......

> defer是在return之前执行的。这个在 官方文档中是明确说明了的。要使用defer时不踩坑，最重要的一点就是要明白，return xxx这一条语句并不是一条原子指令!</br>函数返回的过程是这样的：先给返回值赋值，然后调用defer表达式，最后才是返回到调用函数中。defer表达式可能会在设置函数返回值之后，在返回到调用函数之前，修改返回值，使最终的函数返回值与你想象的不一致。    

## 问题解析
其实使用defer时，用一个简单的转换规则改写一下，就不会迷糊了。改写规则是将return语句拆成两句写，return xxx会被改写成:

* 返回值 = xxx
* 调用defer函数
* 空的return

``` go
//先看例1，它可以改写成这样：
func f() (result int) {
    result = 0 //return语句不是一条原子调用，return xxx其实是赋值＋ret指令
    func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
        result++
    }()
    return
}
//所以这个返回值是1。
 
//再看例2，它可以改写成这样：
func f() (r int) {
    t := 5
    r = t //赋值指令
    func() { //defer被插入到赋值与返回之间执行，这个例子中返回值r没被修改过
        t = t + 5
    }
    return //空的return指令
}
//所以这个的结果是5。
 
//最后看例3，它改写后变成：
func f() (r int) {
    r = 1 //给返回值赋值
    func(r int) { //这里改的r是传值传进去的r，不会改变要返回的那个r值
        r = r + 5
    }(r)
    return //空的return
}
//所以这个例子的结果是1
``` 
## defer的实现
defer关键字的实现跟go关键字很类似，不同的是它调用的是runtime.deferproc而不是runtime.newproc。在defer出现的地方，插入了指令call runtime.deferproc，然后在函数返回之前的地方，插入指令call runtime.deferreturn。

普通的函数返回时，汇编代码类似：
``` go
add xx SP
return
```
如果其中包含了defer语句，则汇编代码是：
``` go
call runtime.deferreturn，
add xx SP
return
```
goroutine的控制结构中，有一张表记录defer，调用runtime.deferproc时会将需要defer的表达式记录在表中，而在调用runtime.deferreturn的时候，则会依次从defer表中出栈并执行。
