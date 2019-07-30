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
	
#### 5
下面代码会输出什么
``` go
package main

import "fmt"

func main() {
	/* 创建切片 */
	numbers := []int{0, 1, 2, 3, 4, 5, 6, 7, 8}
	printSlice(numbers)
	number3 := numbers[2:7]
	printSlice(number3)
	number4 := number3[2:5]
	printSlice(number4)
}


func printSlice(x []int) {
	fmt.Printf("len=%d cap=%d slice=%v\n", len(x), cap(x), x)
}

``` 
答案
``` go
//  注意cap的运算规则
len=9 cap=9 slice=[0 1 2 3 4 5 6 7 8]
len=5 cap=7 slice=[2 3 4 5 6]
len=3 cap=5 slice=[4 5 6]
``` 

写出以下打印结果，并解释下为什么这么打印的。
``` go
package main

import (
	"fmt"
)

func main() {
	str1 := []string{"a", "b", "c"}
	str2 := str1[1:]
	str2[1] = "new"
	fmt.Println(str1)
	str2 = append(str2, "z", "x", "y")
	fmt.Println(str1)
	str2[1] = "old"
	fmt.Println(str1)
}

``` 

``` go
答案：
[a b new]
[a b new]
[a b new]

append 的结果是一个包含原 slice 所有元素加上新添加的元素的 slice。
已经不再是原来的那个slice了
``` 


#### 6
下面代码会输出什么
``` go
package main

import (
	"fmt"
)

type Vertex struct {
	X, Y float64
}

var e = &Vertex{
	X: 11,
	Y: 22,
}

func (v *Vertex) Scale() {
	v = e
	fmt.Println(v)
}

func main() {
	v := Vertex{3, 4}
	v.Scale()
	fmt.Println(v)
}

``` 

答案
``` go
//  reciver 可以修改自身的内容，但是无法将自身指向另一个地址
&{11 22}
{3 4}
``` 

#### 7
下面代码能运行吗？为什么。
``` go
type Param map[string]interface{}

type Show struct {
	Param
}

func main1() {
	s := new(Show)
	s.Param["RMB"] = 10000
}
``` 

答案
``` go
不能
map里如果value是结构体，不能修改其值。因为没法寻址
但如果是结构体指针的话可以
``` 


#### 8
请说出下面的代码存在什么问题？
``` go
type query func(string) string

func exec(name string, vs ...query) string {
	ch := make(chan string)
	fn := func(i int) {
		ch <- vs[i](name)
	}
	for i, _ := range vs {
		go fn(i)
	}
	return <-ch
}

func main() {
	ret := exec("111", func(n string) string {
		return n + "func1"
	}, func(n string) string {
		return n + "func2"
	}, func(n string) string {
		return n + "func3"
	}, func(n string) string {
		return n + "func4"
	})
	fmt.Println(ret)
}
``` 

答案
``` go
大概率是  111func4
但也有可能是其他值

go机制导致大概率为遍历完成后再起协程

``` 


#### 9
请说出下面的代码存在什么问题？
``` go

package main

import (
    "fmt"
)

func main() {
    fmt.Println([...]string{"1"} == [...]string{"1"})
    fmt.Println([]string{"1"} == []string{"1"})
}
``` 

答案
``` go
数组可以比较
切片不能比较
map也不能
``` 

#### 10
请说出下面的代码存在什么问题？
``` go
package main

import (
	"encoding/json"
	"fmt"
)

type Girl struct {
	Name       string `json:"name"`
	DressColor string `json:"dress_color"`
}

func (g Girl) SetColor(color string) {
	g.DressColor = color
}
func (g Girl) JSON() string {
	data, _ := json.Marshal(&g)
	return string(data)
}
func main() {
	g := Girl{Name: "menglu"}
	g.SetColor("white")
	fmt.Println(g.JSON())
}

``` 

答案
``` go
{"name":"menglu","dress_color":""}

g Girl只是形参

改为g *Girl即可

``` 
