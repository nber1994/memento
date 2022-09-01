tmux互斥锁，即如果锁被强占，后来的竞争者需要阻塞至锁被释放为止。

# 热身

## 原子操作

```go
package main

import (
   "fmt"
   "sync"
)

func main() {
   var counter int64
   var wg sync.WaitGroup
   wg.Add(2)
   for i := 0; i < 2; i++ {
      go func() {
         defer wg.Done()
         for j := 0; j < 10000; j++ {
            counter++  //atomic.AddInt64(&counter, 1)
         }
      }()
   }
   wg.Wait()
   fmt.Println(counter)
}
```

结果会是两万吗？

![](assets_sync.mutex源码解析/2022-08-31-18-13-53-image.png)

## 信号量

# 初版锁实现

```go
package sync

import (
    "runtime"
    "sync/atomic"
)


//互斥锁结构
type Mutex struct {
    key  int32
    sema uint32
}

//请求锁
func (m *Mutex) Lock() {
    if atomic.AddInt32(&m.key, 1) == 1 {    //标识加1，如果等于1，成功获取到锁
        return
    }
    runtime.Semacquire(&m.sema)     //否则阻塞等待
}


//释放锁
func (m *Mutex) Unlock() {
    switch v := atomic.AddInt32(&m.key, -1); {  //标识减1
    case v == 0:    //如果等于0，则没有等待者
        return
    case v == -1:   //如果等于-1，这种是异常情况，或者超过了最大可等待goroutine的数量
        panic("sync: unlock of unlocked mutex")
    }
    runtime.Semrelease(&m.sema) //唤醒其他阻塞的goroutine
}
```

## 信号量快速入门

xxxx

这是最初mutex版本的实现，简单来说就是：

- 原子性的对一个变量key加一操作，第一个加一的为锁的持有者
- 剩余的视为未抢到锁，进入休眠
- 而当锁所有者释放锁时，对key进行减一操作，根据操作后key的值来走不同逻辑，
  - 如果key大于0，则说明存在阻塞者，此时只需要唤醒一个等待者即可（Semrelease）
  - 如果等于0则表示没有等待者，直接返回即可

其中，由于信号量中内部会维护一个先入先出队列，则能保证强锁的公平性。

其中，会导致panic的点如下：

- 对一个未上锁的锁进行解锁操作
- 当有过多的竞争者抢锁，int32到达上线后

# 改进版 - 给要抢锁的G多一次机会

我们需要知道一个现状，**G的唤醒与休眠是一个昂贵的操作**，而大部分加锁的临界区都比较小。

也就是说锁很可能很快就释放锁了，如果能让参与抢锁的G稍微再多一次重试，也许在下次重试时，锁已经被释放了，这样就免去了一次G的休眠和唤醒。

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package sync provides basic synchronization primitives such as mutual
// exclusion locks.  Other than the Once and WaitGroup types, most are intended
// for use by low-level library routines.  Higher-level synchronization is
// better done via channels and communication.
package sync

import (
    "runtime"
    "sync/atomic"
)

// A Mutex is a mutual exclusion lock.
// Mutexes can be created as part of other structures;
// the zero value for a Mutex is an unlocked mutex.
type Mutex struct {
    state int32
    sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)

// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }

    awoke := false
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 {
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime.Semacquire(&m.sema)
            awoke = true
        }
    }
}

// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        panic("sync: unlock of unlocked mutex")
    }

    old := new
    for {
        // If there are no waiters or a goroutine has already
        // been woken or grabbed the lock, no need to wake anyone.
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
            return
        }
        // Grab the right to wake someone.
        new = (old - 1<<mutexWaiterShift) | mutexWoken
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime.Semrelease(&m.sema)
            return
        }
        old = m.state
    }
}
```

![26c0382433b84652a559a2dcb8201c82_tplv-k3u1fbpfcp-zoom-in-crop-mark_1304_0_0_02022031117562420220325173047.png](assets_sync.mutex源码解析/2e136e21f786231ba344c7f34d4d071a21ef395b.png)

## 加锁流程

1. 先尝试cas更新状态为已加锁，如果成功
   1. 则直接返回
2. 开启for 循环
   1. 保存当前状态old以备后续57行cas判断使用
   2. 创建一个新的状态new，并且打上加锁标识，以备后续进行更新
   3. 若old状态是已加锁
      1. 则本G则会变成等待着，所以新增new中等待者的值
   4. 如果当前是被唤醒状态
      1. 说明当前G是被唤醒的，则给new清楚调唤醒标识
   5. 进行cas更新状态，以old作为比较值，new作为更新值，如果更新成功
      1. 判断old是否是未加锁的状态，若是
         1. 直接返回，说明我们抢到了锁
      2. 进入休眠，等待被唤醒，代码会停止在这一步
      3. 设置awoke为true，标识当前G被唤醒了
3. 开启下一轮for循环

## 解锁流程

1. 直接对加锁标识位-1
2. 判断变更后的new加锁标识位是否为-1，若是
   1. 则表示对未加锁的锁进行了解锁，直接panic
3. 保存当前锁状态old
4. 开启for循环，尝试唤醒一个等待者
   1. 判断当前锁是否不存在等待者了，或者当前锁已经被加锁了或者唤醒了，若是
      1. 则说明已经不需要我们再唤醒等待者了，直接返回即可，解锁完成
   2. 将状态的等待者数量减一，并且打上唤醒标识，赋值给new
   3. 更新锁状态，old作为比较值，new作为更新值，cas更新，若成功
      1. 调用信号量唤醒一个等待G
      2. 返回
   4. 重新读取当前锁状态到old
5. 开启下一轮for循环

对于加锁者来说，我需要做的就是在第一轮没抢到锁之后，再次尝试去进行抢锁，第二次有两种结果，一个是抢到了锁则直接返回，另一个是仍然没抢到，则我机会用完，直接进入休眠。

对于解锁者来说，我的任务一个是修改锁的加锁状态为未加锁，另外，我还要去尝试更新锁的状态为已唤醒，表示将要唤醒一个等待者，更新成功后，通过信号量唤醒一个G，解锁完毕。

总体来说，这个版本的锁，给了第一次抢锁失败的竞争者一次另外的机会，再去抢一次锁。打破了原来的先到先得的公平性，越靠后的越有可能会有限抢到锁。

# 改进版 v2 - 给要抢锁的G更多的机会

上个版本，给抢锁的G多了一次机会，在一定程度上通过提高了cpu占用率与抢锁的公平性，换到了性能的提升。那么这一版本的改进就是，进一步的提高cpu占用率，看是否能获得更高的性能提升。

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package sync provides basic synchronization primitives such as mutual
// exclusion locks.  Other than the Once and WaitGroup types, most are intended
// for use by low-level library routines.  Higher-level synchronization is
// better done via channels and communication.
//
// Values containing the types defined in this package should not be copied.
package sync

import (
    "sync/atomic"
    "unsafe"
)

// A Mutex is a mutual exclusion lock.
// Mutexes can be created as part of other structures;
// the zero value for a Mutex is an unlocked mutex.
type Mutex struct {
    state int32
    sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)

// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if raceenabled {
            raceAcquire(unsafe.Pointer(m))
        }
        return
    }

    awoke := false
    iter := 0
    for {
        old := m.state
        new := old | mutexLocked
        if old&mutexLocked != 0 {
            if runtime_canSpin(iter) {
                // Active spinning makes sense.
                // Try to set mutexWoken flag to inform Unlock
                // to not wake other blocked goroutines.
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue
            }
            new = old + 1<<mutexWaiterShift
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                panic("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 {
                break
            }
            runtime_Semacquire(&m.sema)
            awoke = true
            iter = 0
        }
    }

    if raceenabled {
        raceAcquire(unsafe.Pointer(m))
    }
}

// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
    if raceenabled {
        _ = m.state
        raceRelease(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if (new+mutexLocked)&mutexLocked == 0 {
        panic("sync: unlock of unlocked mutex")
    }

    old := new
    for {
        // If there are no waiters or a goroutine has already
        // been woken or grabbed the lock, no need to wake anyone.
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 {
            return
        }
        // Grab the right to wake someone.
        new = (old - 1<<mutexWaiterShift) | mutexWoken
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime_Semrelease(&m.sema)
            return
        }
        old = m.state
    }
}
```

这次的优化比较小，大致保留了之前的代码骨架，只是在Lock函数中增加了一个runtime_canSpin的检测，通过自旋的计数器iter，让当前G自旋等待一会，然后再去抢锁，这样加大了抢到锁的概率，同样的是以牺牲cpu使用率来换取了锁的性能。

为了实现多等一会的操作，我们最简单的操作就是sleep或者for循环一定次数，达到等待一阵子的目的，但是结合调度器来看，如果等待的时间过长的话，当前G会被打上抢占标识，可能会被调度让出cpu（这个概率很大），这正是我们希望避免的，所以我们引入了自旋操作，在自旋操作期间，我们可以保证当前G不会被调度出cpu。但是自旋一不小心就会造成死锁问题：

## 自旋的条件

xxxx

我们来看下新增的一些代码

```go
        if old&mutexLocked != 0 {
            if runtime_canSpin(iter) {
                // Active spinning makes sense.
                // Try to set mutexWoken flag to inform Unlock
                // to not wake other blocked goroutines.
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue
            }
            new = old + 1<<mutexWaiterShift
        }
```

通过注释我们可以看到，在自旋开始之前，尝试对state字段打上唤醒标识，并且将自己伪造成刚被唤醒的G（awoke=true），这么做的目的呢，其实也是为了减少性能消耗，唤醒或者休眠都是奢侈操作，既然我当前G自旋，那么不用再唤醒额外的一个G参与竞争了。

# 先停下来分析下

事情往往是这样的，我们希望解决一个问题，想到了一个思路，进行了一些验证性的优化，拿到了一些不错的结果，然后我们会情不自禁的想随着这个思路加大优化力度，企图拿到更好的结果。那么新的问题往往也是在这个过程中产生的:P

![](assets_sync.mutex源码解析/2022-08-31-18-18-13-image.png)

我们分析了三版mutex的代码，也了解了当前的主要优化方向是 【通过提高锁竞争者的CPU持有时间，减少G的唤醒和休眠次数，来提高锁的整体性能】，那么这种优化思路给我们带来了哪些问题：

1. 破坏了竞争者的公平性
   1. 越后来的竞争者越容易获得锁
2. 出现了饥饿问题
   1. 基于第一条，那些已经在信号量等待队列中的G，可能很长时间或者永远也拿不到锁
3. 提高了CPU占用率
   1. 害，这个必须的

# 最终版本 - 横扫饥饿，做回自己

基于上述的问题，最大的就是饥饿问题，信号量等待队列中的G可能会长时间等待甚至一直等待，这就是饥饿问题，为了解决这个问题，锁引入了饥饿模式，**在饥饿模式开启下，新来的竞争者G直接进入信号量等待队列，而不会再自旋。**

我们先来看下代码注释：

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Package sync provides basic synchronization primitives such as mutual
// exclusion locks. Other than the Once and WaitGroup types, most are intended
// for use by low-level library routines. Higher-level synchronization is
// better done via channels and communication.
//
// Values containing the types defined in this package should not be copied.
package sync

import (
    "internal/race"
    "sync/atomic"
    "unsafe"
)

func throw(string) // provided by runtime

// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
    state int32
    sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}

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

    // 公平锁
    //
    // 锁有两种模式：正常模式和饥饿模式。
    // 在正常模式下，所有的等待锁的goroutine都会存在一个先进先出的队列中（轮流被唤醒）
    // 但是一个被唤醒的goroutine并不是直接获得锁，而是仍然需要和那些新请求锁的（new arrivial）
    // 的goroutine竞争，而这其实是不公平的，因为新请求锁的goroutine有一个优势——它们正在CPU上
    // 运行，并且数量可能会很多。所以一个被唤醒的goroutine拿到锁的概率是很小的。在这种情况下，
    // 这个被唤醒的goroutine会加入到队列的头部。如果一个等待的goroutine有超过1ms（写死在代码中）
    // 都没获取到锁，那么就会把锁转变为饥饿模式。
    //
    // 在饥饿模式中，锁的所有权会直接从释放锁(unlock)的goroutine转交给队列头的goroutine，
    // 新请求锁的goroutine就算锁是空闲状态也不会去获取锁，并且也不会尝试自旋。它们只是排到队列的尾部。
    //
    // 如果一个goroutine获取到了锁之后，它会判断以下两种情况：
    // 1. 它是队列中最后一个goroutine；
    // 2. 它拿到锁所花的时间小于1ms；
    // 以上只要有一个成立，它就会把锁转变回正常模式。

    // 正常模式会有比较好的性能，因为即使有很多阻塞的等待锁的goroutine，
    // 一个goroutine也可以尝试请求多次锁。
    // 饥饿模式对于防止尾部延迟来说非常的重要。
    starvationThresholdNs = 1e6
)

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
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}

func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway.
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case.
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
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
            old = m.state
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}

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
    if new != 0 {
        // Outlined slow path to allow inlining the fast path.
        // To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
        m.unlockSlow(new)
    }
}

func (m *Mutex) unlockSlow(new int32) {
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
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
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        // Starving mode: handoff mutex ownership to the next waiter, and yield
        // our time slice so that the next waiter can start to run immediately.
        // Note: mutexLocked is not set, the waiter will set it after wakeup.
        // But mutex is still considered locked if mutexStarving is set,
        // so new coming goroutines won't acquire it.
        runtime_Semrelease(&m.sema, true, 1)
    }
}
```

锁的state中新增了一个是否饥饿的标识位，同时定义了一个1ms的常量作为是否饥饿的判断阈值。

```go
// 公平锁
//
// 锁有两种模式：正常模式和饥饿模式。
// 在正常模式下，所有的等待锁的goroutine都会存在一个先进先出的队列中（轮流被唤醒）
// 但是一个被唤醒的goroutine并不是直接获得锁，而是仍然需要和那些新请求锁的（new arrivial）
// 的goroutine竞争，而这其实是不公平的，因为新请求锁的goroutine有一个优势——它们正在CPU上
// 运行，并且数量可能会很多。所以一个被唤醒的goroutine拿到锁的概率是很小的。在这种情况下，
// 这个被唤醒的goroutine会加入到队列的头部。如果一个等待的goroutine有超过1ms（写死在代码中）
// 都没获取到锁，那么就会把锁转变为饥饿模式。
//
// 在饥饿模式中，锁的所有权会直接从释放锁(unlock)的goroutine转交给队列头的goroutine，
// 新请求锁的goroutine就算锁是空闲状态也不会去获取锁，并且也不会尝试自旋。它们只是排到队列的尾部。
//
// 如果一个goroutine获取到了锁之后，它会判断以下两种情况：
// 1. 它是队列中最后一个goroutine；
// 2. 它拿到锁所花的时间小于1ms；
// 以上只要有一个成立，它就会把锁转变回正常模式。

// 正常模式会有比较好的性能，因为即使有很多阻塞的等待锁的goroutine，
// 一个goroutine也可以尝试请求多次锁。
// 饥饿模式对于防止尾部延迟来说非常的重要。
```
