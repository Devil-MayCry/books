# Go 语言**学习笔记**

#### 

### channel实现原理

#### chan数据结构

`src/runtime/chan.go:hchan`定义了channel的数据结构：

```go
type hchan struct {
    qcount   uint           // 当前队列中剩余元素个数
    dataqsiz uint           // 环形队列长度，即可以存放的元素个数
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 每个元素的大小
    closed   uint32         // 标识关闭状态
    elemtype *_type         // 元素类型
    sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint           // 队列下标，指示元素从队列的该位置读出
    recvq    waitq          // 等待读消息的goroutine队列
    sendq    waitq          // 等待写消息的goroutine队列
    lock     mutex          // 互斥锁，chan不允许并发读写
}
```

从数据结构可以看出channel由队列、类型信息、goroutine等待队列组成，下面分别说明其原理。

##### 2.1 环形队列

chan内部实现了一个环形队列作为其缓冲区，队列的长度是创建chan时指定的。

下图展示了一个可缓存6个元素的channel示意图：

![](https://oscimg.oschina.net/oscnet/f1ae952fd1c62186d4bd0eb3fa1610db67a.jpg)

* dataqsiz指示了队列长度为6，即可缓存6个元素；
* buf指向队列的内存，队列中还剩余两个元素；
* qcount表示队列中还有两个元素；
* sendx指示后续写入的数据存储的位置，取值\[0, 6\)；
* recvx指示从该位置读取数据, 取值\[0, 6\)；

##### 2.2 等待队列

从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞。  
向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞。

被阻塞的goroutine将会挂在channel的等待队列中：

* 因读阻塞的goroutine会被向channel写入数据的goroutine唤醒；
* 因写阻塞的goroutine会被从channel读数据的goroutine唤醒；

下图展示了一个没有缓冲区的channel，有几个goroutine阻塞等待读数据：

![](https://oscimg.oschina.net/oscnet/51d91ed6fb42117d5035cab82b283bf0b07.jpg)

注意，一般情况下recvq和sendq至少有一个为空。只有一个例外，那就是同一个goroutine使用select语句向channel一边写数据，一边读数据。

##### 2.3 类型信息

一个channel只能传递一种类型的值，类型信息存储在hchan数据结构中。

* elemtype代表类型，用于数据传递过程中的赋值；
* elemsize代表类型大小，用于在buf中定位元素位置。

##### 2.4 锁

一个channel同时仅允许被一个goroutine读写，为简单起见，本章后续部分说明读写过程时不再涉及加锁和解锁。

#### 

#### panic



