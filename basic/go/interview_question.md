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
```
打印后
打印中
打印前
panic: 触发异常
``` 
#### 2
以下代码有什么问题

``` go
package main
import (
    "fmt"
)
type student struct {
    Name string
    Age  int
}
func pase_student() map[string]*student {
    m := make(map[string]*student)
    stus := []student{
        {Name: "zhou", Age: 24},
        {Name: "li", Age: 23},
        {Name: "wang", Age: 22},
    }
    for _, stu := range stus {
        m[stu.Name] = &stu
    }
    return m
}
func main() {
    students := pase_student()
    for k, v := range students {
        fmt.Printf("key=%s,value=%v \n", k, v)
    }
}
```
答案:

for循环使用stu遍历时，stu只是一个临时变量，遍历过程中指针地址不变，所以后面的赋值都是指向了同一个内存区域，导致最后所有的信息都一致。
```
key=zhou,value=&{wang 22} 
key=li,value=&{wang 22} 
key=wang,value=&{wang 22}
```

#### 3
下面代码会输出什么
``` go 
type People struct{}
func (p *People) ShowA() {
    fmt.Println("showA")
    p.ShowB()
}
func (p *People) ShowB() {
    fmt.Println("showB")
}
type Teacher struct {
    People
}
func (t *Teacher) ShowB() {
    fmt.Println("teacher showB")
}
func main() {
    t := Teacher{}
    t.ShowA()
}
```
答案

go中没有继承，只有组合。Teacher中的People是一个匿名对象，通过它调用的函数都是自身的。
```
showA
showB
```

#### 4
下面代码会输出什么
``` go
package main

import (
	"fmt"
)

type Name interface {
	Name() string
}

type A struct {
}

func (self A) say() {
	println(self.Name())
}

func (self A) sayReal(child Name) {
	fmt.Println(child.Name())
}

func (self A) Name() string {
	return "I'm A"
}

type B struct {
	A
}

func (self B) Name() string {
	return "I'm B"
}

type C struct {
	A
}

func main() {
	b := B{}
	b.say()      //I'm A   B没有重写say()方法，所以是用的A来调用
	b.sayReal(b) //I'm B

	c := C{}
	c.say()      //I'm A
	c.sayReal(c) //I'm A   C没有实现Name接口
}
```

#### 4
下面代码会输出什么
``` go
type Persion interface {
   say(p []byte)
   do() (bool)
}
type Worker  struct {
 
}
func (w Worker) say(p []byte) {
   fmt.Println("worker has some words to say", p)
}
func (w Worker) do() (bool) {
   fmt.Println("just do it ")
   return true
}
func (w Worker) who() {
   fmt.Println("I am worker")
}
//测试
var p Persion
w := Worker{}
 
p = w //将对象赋值给接口
// p = &w //将对象对应的指针赋值给该接口
 
p.say([]byte("i have too many jobs"))
p.do()
//p.who() //将会报错，因为接口中没有定义该方法
``` 
答案
``` go
结果见注释
go的接口有如下特性：
（1）实现了接口的定义方法的实例，可以赋值给接口；
（2）若接口B定义的方法，在A接口中都有，那么也可以将接口B赋值给接口A。
``` 


如果修改后会输出什么
``` go
func (w *Worker) do() (bool) {
   fmt.Println("just do it ")
   return true
}
``` 
答案
``` go
p = &w //正常执行
p = w //报错
``` 
| 类型 | 方法 |
| ------ | ------ | ------ |
| 值类型(T)  | 值类型的对象仅仅拥有(T)类型的方法|
| 指针类型(*T) |	指针类型的对象，可以拥有(T)和(*T)方法|
	

