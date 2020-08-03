### 0 前言

我们知道，多线程下为了确保数据不会出错，必须加锁后才能访问共享资源。常见的锁包括互斥锁、自旋锁、读写锁等等，往往需要通过系统内核来实现。

然而协程的互斥锁实现原理完全不同，它并不与内核打交道，虽然不能跨线程工作，但却因为减少了用户态与内核态转换时所需的上下文切换环节，效率很高，是一个非常不错的选择。

本文以golang为例，**用flowchart作图结合代码的形式**，讲解在golang中是如何使用用户态的代码将锁重新实现的。



### 1 前提知识点

#### 1.1 互斥锁与自旋锁

我们常见的各种锁是有层级的，最底层的两种锁就是互斥锁和自旋锁，其他锁都是基于它们实现的。互斥锁的加锁成本更高，但它在加锁失败时会释放 CPU 给其他线程；自旋锁则正好相反。

因此在于不同的情况下，正确的选择互斥锁与自旋锁能很大程度的提升速度。

在golang中的mutex是一个混合锁，优先判断是否自旋，不行才会使用互斥锁。



#### 1.2 正常模式与饥饿模式

mutex存在两种状态——正常模式与饥饿模式。

在golang中，休眠的 goroutine 以 FIFO 链表形式保存在 sudog 中，被唤醒的 goroutine 与新到来活跃的 goroutine 竞解，但是很可能会失败，这样会导致一个goutine始终抢不到CPU资源，出现饥饿状态。

解决方法是如果一个 goroutine 等待超过 1ms，那么 Mutex 进入饥饿模式，在饥饿模式下，锁直接交给等待队列（waiter FIFO链表）的第一个，新来的goroutine直接放到FIFO队尾，不会参与竞争，一直到当前的goroutine是FIFO的最后一个就退出饥饿模式



### 2 mutex相关知识点

#### 2.1 mutex的结构

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

Mutex是一个全局对象，包含两个成员，state与sema

sema是一个信号量，用来唤醒goroutine，初始为0，用于判断是否有可用资源。没有的话就一直等待。

state是一个4字节（32位）的变量，由于4部分组成，是锁的本体

(1) 0位判断当前锁是否上锁

(2) 1位判断当前锁是否是被其他goroutine唤醒的

(3) 2位判断当前锁是否处于饥饿状态

(4) 3-31位用于计算当前等待的goroutine数量



![结构](https://raw.githubusercontent.com/lcylichenyi/markdownImage/master/mutex/2020-01-23-15797104328010-golang-mutex-state.png)



#### 2.2 自旋

在golang中的自旋一次就是将寄存器中的值循环减30次

方法：runtime_doSpin()

```go
// dospin定义，调用了procyield方法
active_spin_cnt = 30

func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}

// procyield定义
func procyield(cycles uint32)

// 实现, 即将对ax寄存器放入30，减到0之后就算自旋完成一次返回
TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```





### 3 mutex的流程图

#### 上锁：

**（流程图使用了flowchart进行绘画）**

```flow
start=>start: 开始
end=>end: 结束，完成上锁
record=>operation: 4.记录当前G状态
park=>operation: 7.排队，挂起
iter=>operation: 3.自旋次数+1
wake=>operation: 8.被唤醒
clearSpin=>operation: 10.自旋数清零

isLocked=>condition: 1.有无锁
spin=>condition: 2.是否自旋
getLock=>condition: 5.是否抢到锁
isHungryOrLocked=>condition: 6.当前锁是否上锁或饥饿
isHungry=>condition: 9.当前锁是否饥饿


start->isLocked

isLocked(yes)->spin
isLocked(no)->end


spin(yes)->iter(left)->spin
spin(no)->record->getLock

getLock(yes)->isHungryOrLocked
getLock(no)->spin

isHungryOrLocked(yes)->park->wake->isHungry
isHungryOrLocked(no)->end

isHungry(yes,right)->end
isHungry(no)->clearSpin(left)->spin



```





下图在代码中标志出对应步骤

```go
func (m *Mutex) Lock() {
	// 步骤1，没锁的话直接拿走
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}

	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
    // 步骤2，判断是否自旋
    // (1)在饥饿模式下不应该自旋
    // (2)是否符合能自旋的条件，自旋次数<4，多核，本地runq为空
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
      // 步骤3，自旋次数+1
			iter++
			old = m.state
			continue
		}
		new := old
    
    
    // 步骤4，记录4个当前G的状态，写到new中用于尝试抢到锁后覆盖锁状态
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}

		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
    // 步骤5，是否成功抢到锁并替换锁状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
      // 步骤6，判断当前锁是否上锁或饥饿，如果都没有就直接结束
			if old&(mutexLocked|mutexStarving) == 0 {
				break
			}
			
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
      // 步骤7，排队挂起
			runtime_SemacquireMutex(&m.sema, queueLifo)
      
      // 步骤8，被唤醒
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
      // 步骤9，判断当前锁状态是否饥饿，如果饥饿的话就赶紧结束
			if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
      // 步骤10，不饥饿的话自旋数清零再来一轮判断
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```







#### 解锁：

**（流程图使用了flowchart进行绘画）**

```flow
start=>start: 开始
end=>end: 结束，完成解锁

unLock=>operation: 1.解锁
getLockStatus=>operation: 3.获取锁状态
minusLock=>operation: 5.等待数-1
releaseLock=>operation: 7.释放信号量

isHungry=>condition: 2.是否饥饿
hasWaiting=>condition: 4.是否还有等待的goroutine
getLock=>condition: 6.是否抢到锁

start->unLock->isHungry

isHungry(yes)->releaseLock->end
isHungry(no, down)->getLockStatus->hasWaiting

hasWaiting(no, down)->minusLock->getLock
hasWaiting(yes)->end

getLock(yes)->releaseLock->end
getLock(no, left)->getLockStatus





```





```go
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// 步骤1，解锁
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
  // 步骤2，判断是否饥饿
	if new&mutexStarving == 0 {
    // 步骤3，获取锁状态
		old := new
		for {
      // 步骤4，判断是否还有等待的goroutine(不用释放信号量)
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
      // 步骤5，修改锁状态使等待数-1，尝试用这个锁状态去覆盖
			new = (old - 1<<mutexWaiterShift) | mutexWoken
      // 步骤6，抢锁替换锁状态
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
        // 步骤7，释放信号量
				runtime_Semrelease(&m.sema, false)
				return
			}
      // 步骤3，获取锁状态
			old = m.state
		}
	} else {
	 // 步骤7，释放信号量
		runtime_Semrelease(&m.sema, true)
	}
}
```



