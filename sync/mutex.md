```go
type Mutex struct {
	state int32
	sema  uint32
}

const (
	// b0001 = 1
	mutexLocked = 1 << iota // mutex is locked
	// 2 b0010
	mutexWoken
	// 4 b0100
	mutexStarving // 从state字段中分出一个饥饿标记
	// 从第三位开始存储等待的goroutine
	mutexWaiterShift = iota
	// 1000000 ns = 1ms 一毫秒
	starvationThresholdNs = 1e6
)

func (m *Mutex) Lock() {
	// Fast path: 幸运之路，一下就获取到了锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	// Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	// 当前goroutine等待时间
	var waitStartTime int64
	starving := false // 此goroutine的饥饿标记
	awoke := false    // 唤醒标记
	iter := 0         // 自旋次数
	old := m.state    // 当前的锁的状态
	for {
		// 锁是非饥饿状态，锁还没被释放，尝试自旋
		// mutexLocked                 = b0001
		// mutexStarving               = b0100
		// mutexLocked|mutexStarving   = b0101 （这里第二位和第四位后面的全都是0，剩下决定性的只有第三位了）

		// old&(mutexLocked|mutexStarving) = b0001 那么 第三位必须是0，说明当前状态是非饥饿状态，

		// runtime_canSpin 表示当前可以自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 非唤醒状态，并且没有被唤醒的goroutine，并且有等待的goroutine，尝试给当前状态设置一个唤醒位
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				// 成功设置状态位，则修改
				awoke = true
			}
			// 进行自旋转
			runtime_doSpin()
			// 自旋数量+1
			iter++
			// 再次获得锁状态
			old = m.state // 再次获取锁的状态，之后会检查是否锁被释放了
			continue
		}
		new := old

		// 当前状态是非饥饿状态
		if old&mutexStarving == 0 {
			new |= mutexLocked // 非饥饿状态，加锁
		}

		// mutexLocked|mutexStarving = b0001 | b0100 = b0101
		// 当前状态有锁，或者处于饥饿模式，或者同时处于有锁并且处于饥饿模式
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift // waiter数量加1
		}

		// 当前goroutine处于饥饿状态 并且当前mutex被其他协程持有锁，需要设置饥饿状态位
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving // 设置饥饿状态
		}

		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken // 新状态清除唤醒标记
		}
		// 成功设置新状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {

			// mutexLocked|mutexStarving = b0001 | b0100 = b0101
			// 第一位=0，说明原来的锁被释放，并且 第三位=0，说明不是出于饥饿模式
			// 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// 处理饥饿状态

			// 如果以前就在队列里面，加入到队列头
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			// 阻塞等待
			// queueLifo = true 说明会将当前goroutine是唤醒的，加入队首，
			// queueLifo = false 说明会将当前goroutine是新来的，加入队列尾部，

			// sleep
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)

			// 唤醒之后检查锁是否应该处于饥饿状态
			// 当前时间减去等待开始的时间，是否大于 1ms 毫秒=0.001s
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			// 获取新的状态
			old = m.state

			// 判断当前锁是否处于饥饿状态

			if old&mutexStarving != 0 { // 处于饥饿状态

				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}

				// 有点绕，加锁并且将waiter数减1
				// delta = -7
				delta := int32(mutexLocked - 1<<mutexWaiterShift)

				// 锁仍然处于饥饿状态，但是信唤醒的goroutine，不处于饥饿状态 
				// 或者 只有一个正在等待的goroutine
				if !starving || old>>mutexWaiterShift == 1 {
					// delta = -7 - 4 = -11 = 1b1011
					delta -= mutexStarving // 最后一个waiter或者已经不饥饿了，清除饥饿标记
				}
				// 设置最新的状态
				atomic.AddInt32(&m.state, delta)
				break
			}

			// 未处于饥饿状态，设置唤醒标志
			// 清理自旋次数
			awoke = true
			iter = 0
		} else {
			// CAS不成功，重新获得锁的状态，重新开始循环
			old = m.state
		}
	}
}

func (m *Mutex) Unlock() {
	// Fast path: drop lock bit.
	// 快速解锁 直接将锁位置-1，
	// new !=0 则说明处于饥饿模式，或者有等待的goroutine
	// new ==0 则说明正常模式，并且没有等待者。直接完成
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// 缓慢解锁
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {

	// 解锁一个未加锁的mutex。直接panic
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}

	if new&mutexStarving == 0 { // 处于【正常模式】

		old := new
		for {

			// 没有在等待的协程，或者
			// mutexLocked|mutexWoken|mutexStarving = b0111
			// old & b0111 !=0
			// 说明 有唤醒的goroutine存在 直接结束

			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}

			// 等待数量-1 并且设置唤醒位
			new = (old - 1<<mutexWaiterShift) | mutexWoken

			// 尝试cas设置
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				// 唤醒一个goroutine
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// 处于饥饿模式，直接唤醒一个队列头部的goroutine
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```