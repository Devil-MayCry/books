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
* type 包括 static type和concrete type. 简单来说 static type是你在编码是看见的类型(如int、string)，concrete type是runtime系统看见的类型
* 类型断言能否成功，取决于变量的concrete type，而不是static type. 因此，一个 reader变量如果它的concrete type也实现了write方法的话，它也可以被类型断言为writer.
接下来要讲的反射，就是建立在类型之上的，Golang的指定类型的变量的类型是静态的（也就是指定int、string这些的变量，它的type是static type），在创建变量的时候就已经确定，反射主要与Golang的interface类型相关（它的type是concrete type），只有interface类型才有反射一说。

作者：吴德宝AllenWu
链接：https://juejin.im/post/5a75a4fb5188257a82110544
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。