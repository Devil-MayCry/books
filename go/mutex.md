---
title: Go语言中的锁
date: "2019-05-30"
description: ""
url: /blog/go/mutex/
image: "/blog/go/mutex/title.jpeg"
---
Golang中有两种类型的锁，Mutex （互斥锁）和RWMutex（读写锁）。这里对两种锁的原理进行一个介绍。

<!--more-->

## 基础知识
### 信号量
信号量是 Edsger Dijkstra 发明的数据结构，在解决多种同步问题时很有用。其本质是一个整数，并关联两个操作：
* 申请acquire（也称为 wait、decrement 或 P 操作）
* 释放release（也称 signal、increment 或 V 操作）

acquire操作将信号量减 1，如果结果值为负则线程阻塞，且直到其他线程进行了信号量累加为正数才能恢复。如结果为正数，线程则继续执行。</br>
release操作将信号量加 1，如存在被阻塞的线程，此时他们中的一个线程将解除阻塞。
### 锁的定义
``` go
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}
``` 
在goalng中如果实现了Lock和Unlock方法，那么它就可以被称为锁。

## 读写锁RWMutex
### 结构
首先我们来看看RWMutex大体结构
``` go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
``` 
看到结构发现读写锁内部包含了一个w Mutex互斥锁
注释也很明确，这个锁的目的就是控制多个写入操作的并发执行</br>
writerSem是写入操作的信号量</br>
readerSem是读操作的信号量</br>
readerCount是当前读操作的个数</br>
readerWait当前写入操作需要等待读操作解锁的个数</br>

### 方法
#### RLock（获取读锁）
``` go
// RLock locks rw for reading.
//
// It should not be used for recursive read locking; a blocked Lock
// call excludes new readers from acquiring the lock. See the
// documentation on the RWMutex type.
func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireMutex(&rw.readerSem, false)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}
``` 
先不看竞态检测的部分，先重点中间部分</br>
可以看到，其实很简单，每当有协程需要获取读锁的时候，就将readerCount + 1</br>
但是需要注意的是，这里有一个条件，当readerCount + 1之后的值 < 0的时候，那么将会调用runtime_SemacquireMutex方法</br>

``` go
// Semacquire waits until *s > 0 and then atomically decrements it.
// It is intended as a simple sleep primitive for use by the synchronization
// library and should not be used directly.
func runtime_Semacquire(s *uint32)

// SemacquireMutex is like Semacquire, but for profiling contended Mutexes.
// If lifo is true, queue waiter at the head of wait queue.
func runtime_SemacquireMutex(s *uint32, lifo bool)
``` 
这个方法是一个runtime的方法，会一直等待传入的s出现>0的时候</br>
然后我们可以记得，这里有这样一个情况，当出先readerCount + 1为负数的情况那么就会被等待，看注释我们可以猜到，是当有写入操作出现的时候，那么读操作就会被等待。

#### Lock（获取写锁）
``` go
// Lock locks rw for writing.
// If the lock is already locked for reading or writing,
// Lock blocks until the lock is available.
func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer.
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false)
	}
	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
``` 
写锁稍微复杂一些，但是样子也差不多，我们还是先看中间部分。</br>
首先操作最前面说的互斥锁，目的就是处理多个写锁并发的情况，因为我们知道写锁只有一把。这里不需要深入互斥锁，只需要知道，互斥锁只有一个人能拿到，所以写锁只有一个人能拿到。</br>
</br>
然后重点来了，这里的这个操作细细体会一下，atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders)</br>
是将当前的readerCount减去一个非常大的值rwmutexMaxReaders为1 << 30</br>
大概是1073741823这么大吧</br>
</br>
所以我们可以从源码中看出，readerCount由于每有一个协程获取读锁就+1，一直都是正数，而当有写锁过来的时候，就瞬间减为很大的负数。</br>
然后做完上面的操作以后的r其实就是原来的readerCount。</br>
后面进行判断，如果原来的readerCount不为0（原来有协程已经获取到了读锁）并且将readerWait加上readerCount（表示需要等待readerCount这么多个读锁进行解锁），如果满足上述条件证明原来有读锁，所以暂时没有办法获取到写锁，所以调用runtime_Semacquire进行等待，等待的信号量为writerSem</br>

#### RUnlock（释放读锁）
``` go
// RUnlock undoes a single RLock call;
// it does not affect other simultaneous readers.
// It is a run-time error if rw is not locked for reading
// on entry to RUnlock.
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		if r+1 == 0 || r+1 == -rwmutexMaxReaders {
			race.Enable()
			throw("sync: RUnlock of unlocked RWMutex")
		}
		// A writer is pending.
		if atomic.AddInt32(&rw.readerWait, -1) == 0 {
			// The last reader unblocks the writer.
			runtime_Semrelease(&rw.writerSem, false)
		}
	}
	if race.Enabled {
		race.Enable()
	}
}
``` 
如果是我们来写的话，可能就是将之前+1的readerCount，-1就完事了，但是其实还有一些操作需要注意。</br>
如果-1之后+1==0是啥情况？没错就是我们常见的，新手程序员，没有获取读锁就想去释放读锁，于是异常了。当然+1之后刚好是rwmutexMaxReaders，就证获取了写锁而去释放了读锁，导致异常。</br>
除去异常情况，剩下的就是r还是<0的情况，那么证明确实有协程正在想要获取写锁，那么就需要操作我们前面看到的readerWait，当readerWait减到0的时候就证明没有人正在持有写锁了，就通过信号量writerSem的变化告知刚才等待的协程（想要获取写锁的协程）：你可以进行获取了。</br>
</br>

#### Unlock（释放写锁）
``` go
// Unlock unlocks rw for writing. It is a run-time error if rw is
// not locked for writing on entry to Unlock.
//
// As with Mutexes, a locked RWMutex is not associated with a particular
// goroutine. One goroutine may RLock (Lock) a RWMutex and then
// arrange for another goroutine to RUnlock (Unlock) it.
func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// Announce to readers there is no active writer.
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}
``` 
写锁释放需要恢复readerCount，还记得上锁的时候减了一个很大的数，这个时候要加回来了。</br>
当然加完之后如果>=rwmutexMaxReaders本身，那么还是新手程序员的问题，当没有获取写锁的时候就开始想着释放写锁了。</br>
然后for循环就是为了通知所有在我们RLock方法中看到的，当有因为持有写锁所以等待的那些协程，通过信号量readerSem告诉他们可以动了。</br>
最后别忘记还有一个互斥锁需要释放，让别的协程也可以开始抢写锁了。</br>
</br>
至此，读写锁的分析基本上告一段落了。</br>
针对于其中关于竞态分析的代码，稍后再介绍

## 互斥锁Mutex
``` go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	// Mutex fairness.
	//
	// Mutex can be in 2 modes of operations: normal and starvation.
	// In normal mode waiters are queued in FIFO order, but a woken up waiter
	// does not own the mutex and competes with new arriving goroutines over
	// the ownership. New arriving goroutines have an advantage -- they are
	// already running on CPU and there can be lots of them, so a woken up
	// waiter has good chances of losing. In such case it is queued at front
	// of the wait queue. If a waiter fails to acquire the mutex for more than 1ms,
	// it switches mutex to the starvation mode.
	//
	// In starvation mode ownership of the mutex is directly handed off from
	// the unlocking goroutine to the waiter at the front of the queue.
	// New arriving goroutines don't try to acquire the mutex even if it appears
	// to be unlocked, and don't try to spin. Instead they queue themselves at
	// the tail of the wait queue.
	//
	// If a waiter receives ownership of the mutex and sees that either
	// (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms,
	// it switches mutex back to normal operation mode.
	//
	// Normal mode has considerably better performance as a goroutine can acquire
	// a mutex several times in a row even if there are blocked waiters.
	// Starvation mode is important to prevent pathological cases of tail latency.
	starvationThresholdNs = 1e6
)
``` 
互斥锁有两种操作模式：正常模式和饥饿模式。</br>
在正常模式下等待获取锁的goroutine会以一个先进先出的方式进行排队，但是被唤醒的等待者并不能代表它已经拥有了这个mutex锁，它需要与新到达的goroutine争夺mutex锁。新来的goroutine有一个优势 —— 他们已经在CPU上运行，所以抢到的可能性大一些，所以一个被唤醒的等待者有很大可能抢不过。在这样的情况下，被唤醒的等待者在队列的头部。如果一个被唤醒的等待者抢锁超过1ms失败了，就会切换为饥饿模式。</br>
</br>
在饥饿模式下，mutex锁会直接由解锁的goroutine交给队列头部的等待者。</br>
新来的goroutine不能尝试去获取锁，即使可能根本就没goroutine在持有锁，并且不能尝试自旋。他们只能排到队伍尾巴等着。</br>
</br>
如果一个等待者获取到了锁，并且遇到了下面两种情况之一，就恢复成正常工作模式。</br>
情况1：它是最后一个队列中的等待者。</br>
情况2：它等待的时间小于1ms</br>
</br>
正常模式下，即使有很多阻塞的等待者，有更好的表现，因为一轮能多次获得锁的机会。饥饿模式是为了避免那些一直在队尾的倒霉蛋。</br>

### 基础
#### 结构体
``` go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}
``` 
可以看到，相对读写锁，结构上面很简单，只有两个值，但是千万不要小瞧它，减少了字段就增加了理解难度。</br>
state：将一个32位整数拆分为：</br>
当前阻塞的goroutine数(29位)</br>
饥饿状态(1位，0为正常模式；1为饥饿模式)</br>
唤醒状态(1位，0未唤醒；1已唤醒)</br>
锁状态(1位，0可用；1占用)</br>
sema：信号量</br>


#### CompareAndSwapInt32
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)</br>
这个函数，会先判断参数addr指向的被操作值与参数old的值是否相等，如果相等会将参数new替换参数addr所指向的值，不然的话就啥也不做。</br>
需要特别说明的是，这个方法并不会阻塞。</br>

#### 常量
这是定义的几个常量，我们在一开始的注释周围可以看到，后面需要用到，暂时记住它们的初始值就好。</br>
</br>
mutexLocked = 1 << iota // 1左移0位，是1，二进制是1，（1表示已经上锁）</br>
mutexWoken // 1左移1位，是2，二进制是10</br>
mutexStarving // 1左移2位，是4，二进制是100</br>
mutexWaiterShift = iota // 就是3， 二进制是11</br>
</br>
starvationThresholdNs = 1e6 // 这个就是我们一开始在注释里面看到的1ms，一定超过这个门限值就会更换模式</br>

### Lock获取锁
因为Lock方法比较长，所以我切分一段段看，需要完整的请自己翻看源码。**要注意的一点是，一定要时刻记住，Lock方法是做什么的，很简单，就是要抢锁。看不懂的时候想想这个目标。**

``` go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
``` 
第一步，判断state状态是否为0，如果为0，证明没有协程持有锁，那么就很简单了，直接获取到锁，将mutexLocked（为1）赋值到state就可以了。

``` go
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		// 如果不是饥饿模式，就会进行自旋操作，然后不断循环
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			// awoke表示是否唤醒，old&mutexWoken是取第二位，0表示当前协程未被唤醒，old>>mutexWaiterShift表示右移3位，也就是前29位，不为0证明有协程在等待，并且尝试去对比当前m.state与取出时的old状态，尝试去唤醒自己。然后自旋，并且增加自旋次数表示iter，然后重新赋值old。再循环下一次。
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
``` 
根据注释，这一段大致意思是，如果不是饥饿模式，就会进行自旋操作，然后不断循环。</br>
old&(mutexLocked|mutexStarving) == mutexLocked</br>
（下面均为二进制）</br>
mutexLocked = 1</br>
mutexStarving = 11</br>
mutexLocked = 1</br>
这三个是定值，所以我们容易得到，满足情况的结果为，当old为xxxx0xx（二进制第三位为0）等式成立。</br>
也就是我们一开始说的，state的第三位是表示这个锁当前的模式，0为正常模式，1为饥饿模式。</br>
</br>
那么第一个if就表示，如果当前模式为正常模式，且可以自旋，就进入if条件内部。</br>
</br>
if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
</br>
以上是使用自旋的情况，就是canSpin的。
``` go
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		// 然后进行判断第三位为0的情况，还是所说的正常模式。new就马上拿到锁了，new |= mutexLocked，表示或1，就是第一位无论是啥都赋值为1
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		// 必须当old的1和3两个位置为1的时候才是true，也就是说当前处于饥饿模式，并且锁已经被占用的情况，那么就需要排队去。
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.

    //如果当前已经标记为饥饿模式，并且没有锁住，那么设置new为饥饿模式
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		// 如果唤醒，需要在两种情况下重设标志
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			// 如果唤醒标志为与awoke不相协调就panic
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// 设置唤醒状态位0,被唤醒
			new &^= mutexWoken
		}
``` 
然后进行判断old&mutexStarving == 0就是第三位为0的情况，还是所说的正常模式。new就马上拿到锁了，new |= mutexLocked，表示或1，就是第一位无论是啥都赋值为1</br>
</br>
old&(mutexLocked|mutexStarving)，也就是old & 0101</br>
必须当old的1和3两个位置为1的时候才是true，也就是说当前处于饥饿模式，并且锁已经被占用的情况，那么就需要排队去。</br>
排队也很精妙，new += 1 << mutexWaiterShift</br>
这边注意是先计算1 << mutexWaiterShift也就是将new的前29位+1，就是表示有一个协程在等待了。</br>


``` go
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			// 如果获取锁失败，old刷新状态再次循环，继续cas
			old = m.state
		}
``` 
如果获取锁成功</br>
</br>
old&(mutexLocked|mutexStarving) == 0成立表示已经获取锁，就直接退出CAS</br>
</br>
中间这一段我就不多解释了，就是最前面注释说的，满足什么条件转换什么模式，不多说了。然后从队列中，也就是前29位-1。</br>
需要注意其中有一个runtime_SemacquireMutex和之前看的的runtime_Semacquire是一个意思，只是多了一个参数。</br>

### UnLock释放锁
```  go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
```  
Unlock就相对简单一些，竞态分析不看。</br>
其实我们自己想也能想到，unlock就是将标识位改回来嘛。</br>
然后因为我们已经看过读写锁了，也是同样的道理，如果没有上锁就直接解锁，那肯定报错嘛。</br>
```  go
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false)
				return
			}
			old = m.state
		}
```  
然后如果是正常模式，如果没有等待的goroutine或goroutine已经解锁完成的情况就直接返回了。如果有等待的goroutine那就通过信号量去唤醒runtime_Semrelease（注意这里是false），同时操作一下队列-1</br>
```  go
	} else {
		// Starving mode: handoff mutex ownership to the next waiter.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true)
	}
```  
如果是饥饿模式就直接唤醒（注意这里是true），反正有队列嘛。
## 总结
其实话说回来，我们其实看起来也简单，没有冲突的情况下，能拿就拿呗，如果出现冲突了就尝试自旋解决（自旋一般都能解决）如果解决不了就通过信号量解决，同时如果正常模式就是我们说的抢占式，非公平，如果是饥饿模式，就是我们说的排队，公平，防止有一些倒霉蛋一直抢不到。</br>
</br>
整体总结一下，看完源码我们发现，其实锁的设计并不复杂，主要设计我们要学到cas和处理读写状态的信号量通知，对于那些位操作，能看懂，学可能一时半会学不会，因为很难在一开始就设计的那么巧妙，你也体会到了只用一个变量就维护了整个体系是一种艺术。</br>