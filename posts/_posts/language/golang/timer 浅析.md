---
title: timer 源码分析
date: 2021-11-05
comments: true
---

golang timer 底层实现

<!--more-->

### 环境信息

- go version：go1.8
- compute: linux, centos7

### 带着问题出发
1.数据结构-堆，有什么特征？
2.timer 在执行用户函数的时候是否新建了 goroutine？
3.当 timer 需要阻塞时，g 被谁接管了？
4.timer 中有两种不同的挂起方式分别是？

### timer 中的“增删改查”

#### timer 的创建
```go
// 可以理解为 timer 的入口
func startTimer(t *timer) {
    if raceenabled {
        racerelease(unsafe.Pointer(t))
    }
    // 将新的 timer 加入到全局的 timers 中
    addtimer(t)
}
```
#### addtimer
了解添加 timer 的过程：
```go
/*
    再次说明版本!! go1.8 !!
*/

// 向全局 timers 中添加 timer
func addtimer(t *timer) {
    // 要访问全局的 timers 加锁
    lock(&timers.lock)
    // 将 t 加入到 timers 中
    addtimerLocked(t)
    // 添加完成，释放锁
    unlock(&timers.lock)
}

func addtimerLocked(t *timer) {
    // when must never be negative; otherwise timerproc will overflow
    // during its delta calculation and never expire other runtime timers.
    if t.when < 0 {
        t.when = 1<<63 - 1
    }
    // 计算 t 在 timers 中的索引
    t.i = len(timers.t)
    timers.t = append(timers.t, t)
    // 对小顶堆重新排列
    siftupTimer(t.i)
    // 如果说 t.i == 0，意味着在所有等待执行的 timer 中，这个 timer 的值最小，排在了堆顶
    if t.i == 0 {
        // siftup moved to top: new earliest deadline.
        // 如果 timers 处于 sleeping 状态，则通过 notewakeup 进行唤醒
        if timers.sleeping {
            timers.sleeping = false
            notewakeup(&timers.waitnote)
        }
        // 如果 timers 处于 rescheduling 状态，则通过 goready 进行唤醒
        if timers.rescheduling {
            timers.rescheduling = false
            goready(timers.gp, 0)
        }
    }
    // 检查全局唯一的 timerproc 是否已经创建，没有则启动
    if !timers.created {
        timers.created = true
        go timerproc()
    }
}
```
我们暂时可以不用理解为什么有 sleeping, rescheduling 两种状态，先掌握整体的执行过程，再深入的去看细节。
​

#### timer 的删除
```go
// Delete timer t from the heap.
// Do not need to update the timerproc: if it wakes up early, no big deal.
func deltimer(t *timer) bool {
    // Dereference t so that any panic happens before the lock is held.
    // Discard result, because t might be moving in the heap.
    _ = t.i

    lock(&timers.lock)
    // t may not be registered anymore and may have
    // a bogus i (typically 0, if generated by Go).
    // Verify it before proceeding.
    i := t.i
    last := len(timers.t) - 1
    if i < 0 || i > last || timers.t[i] != t { // 边界检查，校验 timer 是否有效
        unlock(&timers.lock)
        return false
    }
    if i != last {    // 如果要删除的 timer 不是堆中的 最后一个 timer
        timers.t[i] = timers.t[last]    // 堆顶元素与堆尾元素进行互换
        timers.t[i].i = i
    }
    timers.t[last] = nil        // 上述操作已经将要删除的元素放到堆的末尾，直接通过置为 nil 的方式进行删除
    timers.t = timers.t[:last]
    if i != last {              // 重新对堆进行排列
        siftupTimer(i)
        siftdownTimer(i)
    }
    unlock(&timers.lock)        // 删除并重排序完成，释放掉timer的全局锁
    return true
}

```


#### timerproc
addtimer 是生产者的话，我们可以把 timerproc 理解为消费者。
```go
func timerproc() {
    // 注意这里！将执行timerproc的 goroutine 赋值给了 timers 中的 gp 字段。
    timers.gp = getg()
    for {
        lock(&timers.lock) 		
        // 当前执行 timerproc 所以不再是 sleeping 状态，置为 false
        timers.sleeping = false
        now := nanotime()
        // delta 表示 timer 要执行的时间与当前时间的差值，
        // 如果 delta > 0 说明还没有到 timer 的执行时间
        delta := int64(-1)
        for {
            // 判断 timers 中是否有要执行的 timer
            if len(timers.t) == 0 {
                delta = -1
                break
            }
            // 有要执行的，直接获取堆顶元素，小顶堆嘛，堆顶的肯定是最小的
            t := timers.t[0]
            // 计算 delta 差值
            delta = t.when - now
            if delta > 0 {
                break
            }
            // 如果是一个要周期执行的 timer，计算下次执行时间，并重新排列堆
            if t.period > 0 {
                // leave in heap but adjust next time to fire
                t.when += t.period * (1 + -delta/t.period)
                siftdownTimer(0)
            } else {
                // remove from heap
                last := len(timers.t) - 1
                if last > 0 {
                    timers.t[0] = timers.t[last]
                    timers.t[0].i = 0
                }
                timers.t[last] = nil
                timers.t = timers.t[:last]
                if last > 0 {
                    siftdownTimer(0)
                }
                t.i = -1 // mark as removed
            }
            // 获取该 timer 中的函数
            f := t.f
            arg := t.arg
            seq := t.seq
            // 因为执行用户的代码并不需要加锁，所以在此处对锁进行释放，防止其他获取锁的g阻塞太久时间
            unlock(&timers.lock)
            if raceenabled {
                raceacquire(unsafe.Pointer(t))
            }
            // 执行用户函数
            f(arg, seq)
            lock(&timers.lock)
        }
        if delta < 0 || faketime > 0 {	
            // No timers left - put goroutine to sleep.
            // 这种情况意味着 timers 中没有任何可以执行的 timer，gopark挂起，后面goready唤醒
            timers.rescheduling = true
            goparkunlock(&timers.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
            // 恢复回来回到头部执行
            continue
        }
        // At least one timer pending. Sleep until then. timers 中至少有一个可以执行的 timer
        // 设置为 sleeping 我们不使用 gopark 挂起是因为，可能某个timer要执行了，但是你 goready 后重新调度后，会导致这个timer超时
        timers.sleeping = true
        noteclear(&timers.waitnote)
        unlock(&timers.lock)
        // 通过 futex 系统调用，让这个 timerproc 睡一会，醒来之后继续干活
        // 该过程涉及到 handoffp，文章末尾做了简要总结
        notetsleepg(&timers.waitnote, delta)
    }
}
```


至此我们已经解答了文章开头的一部分问题。
​


- 1.在堆的小节去了解
- 2.创建 timer 的时
```go
type runtimeTimer struct {
    i      int			    // 在堆中的位置
    when   int64			// 某个时间点
    period int64			// 周期
    f      func(interface{}, uintptr) // NOTE: must not be closure go的封装的一些函数
    arg    interface{}		// 这个才是我们自己的函数
    seq    uintptr			// 这个好像是没用到
}
```
```go
func goFunc(arg interface{}, seq uintptr) {
    go arg.(func())()
}
```
我们如果写了一个死循环并不会影响timerproc的执行，底层为我们新建了一个 goroutine 去执行。
​


- 3.g 被谁接管了？我在一开始看的时候也有点迷惑，要是能被接管肯定是放在某个地方了，看的时候就是没找到，后来再梳理的时候发现，在 timerproc 函数开始的第一行代码就是 把 g 放入了 timer.gp 字段。
- 4.timer中的两种挂起方式，这个在代码中注释的已经很详细了

1> futex syscall
2> gopark


我们已经看完了整个 创建 timer 的过程，并且知道了 timerproc 就是用来真正执行 timer 的函数。总结一下现在的 timer 有什么问题：
​

**全局只有一把锁，只要对四叉堆修改，必须要加锁，所以怎么做优化才能降低锁的竞争，与此同时，一个非常隐蔽的，也是至关重要的问题正在发生。当我们 timers 中有待执行的对象时，在 timerproc 中通过 syscall 使当前 M 进入阻塞，调度模型中的 handoffp 会将 M 从当前 P 上剥离，然后找一个空闲的 M 或 新建一个 M 执行本地队列中的 g。当进入阻塞的 syscall 调用完成了的时候，需要把 timerproc 重新放回队列进行调度。这种频繁的上下文切换白白浪费了资源。**（在 1.14 版本中去掉了 timerproc 把执行 timer 的逻辑嵌入到调度循环，sysmon等函数中）


### 实际开发中的 timer
#### time.Sleep() 后发生了什么？
日常的开发工作中，或者是写 demo 的时候，经常会用到 time.Sleep()，会让当前的 goroutine 睡一会，等到满足某个 condition 的时候再醒来继续工作。
废话不多说，直接上源码。
```go
// timeSleep puts the current goroutine to sleep for at least ns nanoseconds.
//go:linkname timeSleep time.Sleep 可以看到当我们使用 time.Sleep 的时候实际上底层调用的是 timeSleep() 这个函数
func timeSleep(ns int64) {
    if ns <= 0 {
        return
    }

    // 新建一个 timer 对象
    t := new(timer)
    // 计算要让这个 goroutine 休眠的时间
    t.when = nanotime() + ns
    // t.f 前文中有提到过，是到时后要执行的那个 function
    t.f = goroutineReady
    // 上述 function 的 arg，把执行 timeSleep 的 goroutine 存储到 t.arg 中
    t.arg = getg()
    // 新的 timer 封装好了，要把新 timer 加入到 堆中，保证并发安全，上锁！
    lock(&timers.lock)
    addtimerLocked(t)			
    // 挂起当前的 goroutine
    goparkunlock(&timers.lock, "sleep", traceEvGoSleep, 2)
}

```
#### time.AfterFunc() 后发生了什么？
经过上述的分析是不是已经很简单了，不过我们还是来看一下源码。
```go
// AfterFunc waits for the duration to elapse and then calls f
// in its own goroutine. It returns a Timer that can
// be used to cancel the call using its Stop method.
func AfterFunc(d Duration, f func()) *Timer {
    t := &Timer{    // 封装 timer
        r: runtimeTimer{
            when: when(d),
            f:    goFunc,
            arg:  f,
        },
    }
    // startTimer 前文中提到过，内部就是将新的timer加入到堆中，其中逻辑前文都覆盖到了
    startTimer(&t.r)
    return t
}
```


### 堆的简单了解
go1.8 版本的 timers 结构如下图所示，全局共享一个四叉堆，那么当访问这个数据结构的时候，为了保证并发的安全性，一定是要获取到锁的。
也正是因此锁成了 go1.8 中 timer 主要的性能瓶颈。
![image.png](https://gitee.com/yangbaoqiang/images/raw/f4fa80bedef662b0abaf2a695cefcc0658c83439/blogpics/1635754910137-44029b08-6db8-4417-a709-0ff8b7b38a24.png)


我们简单说一下堆这个数据结构，二叉树大家都很了解，那怎么把一颗二叉树转换成一个二叉堆呢？go 中存储 timer 的数据结构就是一个切片，那什么样的树结构能够使用数组进行存储呢？
​

这里先给出结论：
1.完全二叉树可以使用数组来存储。
2.当一颗二叉树满足完全二叉树的性质，并且满足每个结点的值都小于或等于其左右孩子结点的值，称为小顶堆，大顶堆概念相反。


如上图所示，四叉堆，各个元素存储在数组中，如果想得到子节点的父节点，设子节点索引为 i，那么父节点 p = (i - 1) / 4 (图上的n画错了应该是 i，懒得改了）。
​

#### timer 四叉堆排序算法
```go
// Heap maintenance algorithms.
// 由下向上
// 主要的算法思想是：用索引为 i 的timer，不断地找到该 timer 的 parent，并比较两者的 when 进行交换
func siftupTimer(i int) {
    t := timers.t
    when := t[i].when					
    tmp := t[i]
    for i > 0 {
        p := (i - 1) / 4 // parent
        if when >= t[p].when {
            break
        }
        t[i] = t[p]
        t[i].i = i
        t[p] = tmp
        t[p].i = p
        i = p
    }
}

// 由上向下
func siftdownTimer(i int) {
    t := timers.t
    n := len(t)    // 堆中总的元素数量
    when := t[i].when
    tmp := t[i]
    for {
        c := i*4 + 1 // left child
        c3 := c + 2  // mid child
        if c >= n {	// 判断 i 的四个子节点中的第一个是否超过了总节点个数
            break
        }
        w := t[c].when 
        if c+1 < n && t[c+1].when < w { // 找到前两个子节点中最小的
            w = t[c+1].when
            c++
        }
        if c3 < n {
            w3 := t[c3].when
            if c3+1 < n && t[c3+1].when < w3 {	// 找到后两个子节点中最小的
                w3 = t[c3+1].when
                c3++
            }
            if w3 < w { // 找到这两组子结点中最小的
                w = w3
                c = c3
            }
        }
        if w >= when {	// 判断子节点中的值是否大于父节点，是就不需要交换
            break
        }
        // 由此开始，是把子节点与父节点进行互换
        t[i] = t[c]
        t[i].i = i
        t[c] = tmp
        t[c].i = c
        i = c
    }
}
```
#### 扩展
如果完全二叉树，采用数组进行存储，索引从 0 开始存储和索引从 1 开始存储有什么区别？可以动手画一下。
### timer 中的上下文切换 
#### notesleepg
在 timers 中没有任何将要执行的 timer 时，会通过 futex 这个系统调用将当前这个 M 挂起，看看代码中是怎么做的。
```go
func notetsleepg(n *note, ns int64) bool {
    gp := getg()
    if gp == gp.m.g0 {
        throw("notetsleepg on g0")
    }
    // 进入系统调用
    entersyscallblock()	
    // 阻塞当前线程
    ok := notetsleep_internal(n, ns)
    // 退出系统调用
    exitsyscall()
    return ok
}
```
```go
func entersyscallblock() {
    (...) // 省略了一些处理 g 状态的代码

    // 切换到 g0 栈，执行 handoff 操作
    systemstack(entersyscallblock_handoff)

    (...)
}
```
```go
func entersyscallblock_handoff() {
    (...) // 省略 trace
    // releasep 中解绑了 M 与 P 的关系
    // handoffp 中会获取线程
    handoffp(releasep())
}
```
```go
// 上述函数将 M 与 P 解绑之后，M 通过 futex 系统调用休眠 ns 
func notetsleep_internal(n *note, ns int64) bool {
    gp := getg()

    if ns < 0 {
        if *cgo_yield != nil {
            // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
            ns = 10e6
        }
        for atomic.Load(key32(&n.key)) == 0 {
            gp.m.blocked = true
            futexsleep(key32(&n.key), 0, ns)
            if *cgo_yield != nil {
                asmcgocall(*cgo_yield, nil)
            }
            gp.m.blocked = false
        }
        return true
    }

    if atomic.Load(key32(&n.key)) != 0 {
        return true
    }

    deadline := nanotime() + ns
    for {
        if *cgo_yield != nil && ns > 10e6 {
            ns = 10e6
        }
        gp.m.blocked = true		// 表示当前 M 正在阻塞
        futexsleep(key32(&n.key), 0, ns)	// 让 M 睡一会
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false	// 休眠完成，修改回 M 的阻塞状态
        if atomic.Load(key32(&n.key)) != 0 {
            break
        }
        now := nanotime()
        if now >= deadline {
            break
        }
        ns = deadline - now
    }
    return atomic.Load(key32(&n.key)) != 0
}
```
M 从系统调用中恢复过来，在缺少 P 的情况下是怎么个执行过程
```go
func exitsyscall() {
    (...)

    oldp := _g_.m.oldp.ptr()    // 获取之前执行这个 M 的 P
    _g_.m.oldp = 0
    // fast path，如果能获取到之前的 P 还是放在原来的 P 上进行调度，否则看看有没有空闲的 P
    if exitsyscallfast(oldp) {
        if _g_.m.mcache == nil {
            throw("lost mcache")
        }
        if trace.enabled {
            if oldp != _g_.m.p.ptr() || _g_.m.syscalltick != _g_.m.p.ptr().syscalltick {
                systemstack(traceGoStart)
            }
        }
        // There's a cpu for us, so we can run.
        _g_.m.p.ptr().syscalltick++
        // We need to cas the status and scan before resuming...
        casgstatus(_g_, _Gsyscall, _Grunning)

        // Garbage collector isn't running (since we are),
        // so okay to clear syscallsp.
        _g_.syscallsp = 0
        _g_.m.locks--
        if _g_.preempt {
            // restore the preemption request in case we've cleared it in newstack
            _g_.stackguard0 = stackPreempt
        } else {
            // otherwise restore the real _StackGuard, we've spoiled it in entersyscall/entersyscallblock
            _g_.stackguard0 = _g_.stack.lo + _StackGuard
        }
        _g_.throwsplit = false

        if sched.disable.user && !schedEnabled(_g_) {
            // Scheduling of this goroutine is disabled.
            Gosched()
        }

        return
    }

    (...)

    // Call the scheduler.
    mcall(exitsyscall0) // slow path put g to globalrunq
    (...)

}
```


感谢你能够看完，笔者能力有限，有问题的地方欢迎大家指出，共同探讨，一起学习、进步！
