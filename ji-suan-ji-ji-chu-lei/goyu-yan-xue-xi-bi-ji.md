# Go 语言**学习笔记**

#### 

### channel实现原理

#### chan数据结构

`src/runtime/chan.go:hchan`定义了channel的数据结构：

``` go
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
#### panic



