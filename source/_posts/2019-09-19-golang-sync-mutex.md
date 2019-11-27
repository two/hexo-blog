title: golang sync.mutex
date: 2019-09-19 15:58:18
tags: [Go]
---
##
##
##

```go
const (
    mutexLocked = 1 << iota     // 1 = 0b001
    mutexWoken                  // 2 = 0b010
    mutexStarving               // 4 = 0b100
    mutexWaiterShift = iota     // 3  用来屏蔽低三位，取数量
    starvationThresholdNs = 1e6 // 10^6 ns = 1 ms
)
```

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

```
// state
   +-----------------------------+---+---+---+                          
   |00000000000000000000000000000|0/1|0/1|0/1|                          
   +-----------------------------+---+---+---+                          
                  |                |   |   |                            
                  |                |   |   |        +-----------+       
                  |                |   |   +------->|mutexLocked|       
                  |                |   |            +-----------+       
                  |                |   |            +----------+        
                  |                |   +----------->|mutexWoken|        
                  |                |                +----------+        
                  |                |                +-------------+     
                  |                +--------------->|mutexStarving|     
                  |                                 +-------------+     
                  |                                 +------------------+
                  +-------------------------------->| wait list count  |
                                                    +------------------+
                                                                   
```


```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64 // 开始等待锁的时间点
    starving := false // 当前 goroutine 是否处于饥饿状态
    awoke := false    // 当前 goroutine 是否被唤醒
    iter := 0         //
    old := m.state    // 当前锁的状态
    for {
        // Don't spin in starvation mode, ownership is handed off to waiters
        // so we won't be able to acquire the mutex anyway.
        // 饥饿模式不要自旋， 因为锁的所有权回直接交给 waiters, 所以我们不会获取到锁
        // 条件翻译为伪代码: isLocked() && isNotStarving() && canSpin() 
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Try to set mutexWoken flag to inform Unlock
            // to not wake other blocked goroutines.
            //  尝试设置 mutexWoken  标志来通知 Unlock 不唤醒其它被阻塞的 goroutine
            //  条件可以转换为: 当前 goroutine 没有被唤醒     && 
                                锁状态没有被唤醒              && 
                                等待获取锁的 goroutine 不为 0 && 
                                锁的状态改从未唤醒更新为 被唤醒
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true // 设置当前 goroutine 为唤醒状态
            }
            runtime_doSpin() // 进入自旋
            iter++
            old = m.state // 更新锁状态
            continue
        }

        // 经过上一步后，锁和状态的组合有下面几个:
        // 获取锁   + 正常模式
        // 获取锁   + 饥饿模式
        // 未获取锁 + 正常模式
        // 未获取锁 + 饥饿模式
        new := old
        // Don't try to acquire starving mutex, new arriving goroutines must queue.
        // 正常模式: 期望设置为获取锁
        // 如果是饥饿模式, 新来的 goroutine 必须放到锁队列尾部排队
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        // (锁被获取 || 饥饿模式): 等待锁的 goroutine 数量 +1
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // The current goroutine switches mutex to starvation mode.
        // But if the mutex is currently unlocked, don't do the switch.
        // Unlock expects that starving mutex has waiters, which will not
        // be true in this case.
        // 当前 gorutine 是（饥饿状态 && 锁被获取): 期望设置锁状态为饥饿模式
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        // 当前 goroutine 处于被唤醒状态
        if awoke {
            // The goroutine has been woken from sleep,
            // so we need to reset the flag in either case.
            // 如果锁状态为被唤醒状态，证明存在冲突
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            //  new 期望设置为非被唤醒状态
            new &^= mutexWoken
        }
        // 更新锁状态为  从 old 变为 new
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // old 原来锁状态不是被获取 && 锁状态不是饥饿状态
            // 根据前面的条件 new 现在是获取锁状态
            // old 和 new 交换成功，所以当前 goroutine 获取到了锁, 直接返回
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // 走到这里: old 是被获取 || old 是饥饿状态
            // waitStartTime != 0 证明等待过, 否则未等待过
            // If we were already waiting before, queue at the front of the queue.
            queueLifo := waitStartTime != 0 // true or false
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 如果等待过则放到锁队列头
            // 否则放到锁队列尾部
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            // 如果等待时间超过了 starvationThresholdNs (1ms), 则设置当前 goroutine 为饥饿模式
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            // 如果原来处于饥饿模式
            if old&mutexStarving != 0 {
                // If this goroutine was woken and mutex is in starvation mode,
                // ownership was handed off to us but mutex is in somewhat
                // inconsistent state: mutexLocked is not set and we are still
                // accounted as waiter. Fix that.
                // 如果当前 goroutine 处于饥饿模式, 但是 mutex出一些冲突的状态: mutexLocked 状态没有设置，当前 goroutine 仍处于 waiter 中
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                // 当前 goroutine 不是饥饿状态 || 等待的 gorouine == 1, 退出饥饿模式
                if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式
                    // Exit starvation mode.
                    // Critical to do it here and consider wait time.
                    // Starvation mode is so inefficient, that two goroutines
                    // can go lock-step infinitely once they switch mutex
                    // to starvation mode.
                    delta -= mutexStarving
                }
                //  等待队列数量 -1 &&  获取锁
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state // 更新 state
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```

```
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

	var waitStartTime int64 // 用来存当前goroutine等待的时间
	starving := false       // 用来存当前goroutine是否饥饿
	awoke := false          // 用来存当前goroutine是否已唤醒
	iter := 0               // 用来存当前goroutine的循环次数(想一想一个goroutine如果循环了2147483648次咋办……)
	old := m.state          // 复制一下当前锁的状态
	for { // 自旋
		// 如果是饥饿情况之下，就不要自旋了，因为锁会直接交给队列头部的goroutine
		// 如果锁是被获取状态，并且满足自旋条件（canSpin见后文分析），那么就自旋等锁
		// 伪代码：if isLocked() && isNotStarving() && canSpin()
        // old&(mutexLocked|mutexStarving) == mutexLocked  满足的条件为: (0x1) & (101) ; 不满足的条件为: (1xx) & (101) 或 (0x1) && (101)
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 将自己的状态以及锁的状态设置为唤醒，这样当Unlock的时候就不会去唤醒其它被阻塞的goroutine了
            // 自己为未唤醒状态, 锁状态为未唤醒, 等待锁的goroutine 数量不为0, 将锁状态从未唤醒更新为唤醒
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true // 当前 goroutine 状态更新为唤醒
			}
			runtime_doSpin() // 进行自旋(分析见后文)
			iter++
			old = m.state // 更新锁的状态(有可能在自旋的这段时间之内锁的状态已经被其它goroutine改变)
			continue
		}
		
		// 当走到这一步的时候，可能会有以下的情况：
		// 1. 锁被获取+ 饥饿
		// 2. 锁被获取+ 正常
		// 3. 锁空闲 + 饥饿
		// 4. 锁空闲 + 正常
		
		// goroutine的状态可能是唤醒以及非唤醒
		
		// 复制一份当前的状态，目的是根据当前状态设置出期望的状态，存在new里面，
		// 并且通过CAS来比较以及更新锁的状态
		// old用来存锁的当前状态
		new := old

		// 如果说锁不是饥饿状态，就把期望状态设置为被获取(获取锁)
		// 也就是说，如果是饥饿状态，就不要把期望状态设置为被获取
		// 新到的goroutine乖乖排队去
		// 伪代码：if isNotStarving()
		if old&mutexStarving == 0 {
			// 伪代码：newState = locked
			new |= mutexLocked
		}
		// 如果锁是被获取状态，或者饥饿状态
		// 就把期望状态中的等待队列的等待者数量+1(实际上是new + 8)
		// (会不会可能有三亿个goroutine等待拿锁……)
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// 如果说当前的goroutine是饥饿状态，并且锁被其它goroutine获取
		// 那么将期望的锁的状态设置为饥饿状态
		// 如果锁是释放状态，那么就不用切换了
		// Unlock期望一个饥饿的锁会有一些等待拿锁的goroutine，而不只是一个
		// 这种情况下不会成立
		if starving && old&mutexLocked != 0 {
			// 期望状态设置为饥饿状态
			new |= mutexStarving
		}
		// 如果说当前goroutine是被唤醒状态，我们需要reset这个状态
		// 因为goroutine要么是拿到锁了，要么是进入sleep了
		if awoke {
			// 如果说期望状态不是woken状态，那么肯定出问题了
			// 这里看不懂没关系，wake的逻辑在下面
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			// 这句就是把new设置为非唤醒状态
			// &^的意思是and not
			new &^= mutexWoken
		}
		// 通过CAS来尝试设置锁的状态
		// 这里可能是设置锁，也有可能是只设置为饥饿状态和等待数量
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			// 如果说old状态不是饥饿状态也不是被获取状态
			// 那么代表当前goroutine已经通过CAS成功获取了锁
			// (能进入这个代码块表示状态已改变，也就是说状态是从空闲到被获取)
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// 如果之前已经等待过了，那么就要放到队列头
			queueLifo := waitStartTime != 0
			// 如果说之前没有等待过，就初始化设置现在的等待时间
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			// 既然获取锁失败了，就使用sleep原语来阻塞当前goroutine
			// 通过信号量来排队获取锁
			// 如果是新来的goroutine，就放到队列尾部
			// 如果是被唤醒的等待锁的goroutine，就放到队列头部
			runtime_SemacquireMutex(&m.sema, queueLifo)
			
			// 这里sleep完了，被唤醒
			
			// 如果当前goroutine已经是饥饿状态了
			// 或者当前goroutine已经等待了1ms（在上面定义常量）以上
			// 就把当前goroutine的状态设置为饥饿
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			// 再次获取一下锁现在的状态
			old = m.state
			// 如果说锁现在是饥饿状态，就代表现在锁是被释放的状态，当前goroutine是被信号量所唤醒的
			// 也就是说，锁被直接交给了当前goroutine
			if old&mutexStarving != 0 {
				// 如果说当前锁的状态是被唤醒状态或者被获取状态，或者说等待的队列为空
				// 那么是不可能的，肯定是出问题了，因为当前状态肯定应该有等待的队列，锁也一定是被释放状态且未唤醒
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				// 当前的goroutine获得了锁，那么就把等待队列-1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 如果当前goroutine非饥饿状态，或者说当前goroutine是队列中最后一个goroutine
				// 那么就退出饥饿模式，把状态设置为正常
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				// 原子性地加上改动的状态
				atomic.AddInt32(&m.state, delta)
				break
			}
			// 如果锁不是饥饿模式，就把当前的goroutine设为被唤醒
			// 并且重置iter(重置spin)
			awoke = true
			iter = 0
		} else {
			// 如果CAS不成功，也就是说没能成功获得锁，锁被别的goroutine获得了或者锁一直没被释放
			// 那么就更新状态，重新开始循环尝试拿锁
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```
