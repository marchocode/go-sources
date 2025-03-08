```go
type RWMutex struct {
	w           Mutex        // 互斥锁
	writerSem   uint32       // writer信号量
	readerSem   uint32       // reader信号量
	readerCount atomic.Int32 // 持有w的reader数量 readerCount 有一个负数的隐藏含义，如果是负数，则表示有writer正在请求锁，
	readerWait  atomic.Int32 // writer获得锁，需要等待reader的数量
}

func (rw *RWMutex) RLock() {

    // ignore race

    // readerCount 是一个负数，表示正在有writer请求锁，当前的goroutine加入等待队列尾部
	if rw.readerCount.Add(1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0)
        // 阻塞醒过来，说明writer执行完成，就获得锁了
	}

    // 获得锁，add 1完成

}

func (rw *RWMutex) RUnlock() {

    // reader 计数器 -1.    
	if r := rw.readerCount.Add(-1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}

    // -1正常，没有小于0. 正常释放锁

}

func (rw *RWMutex) rUnlockSlow(r int32) {

    // 一些错误情况，比如解锁一个未上锁的RWLock
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		fatal("sync: RUnlock of unlocked RWMutex")
	}
    
	// 说明有waiter等待，等待数量-1
	if rw.readerWait.Add(-1) == 0 {
		// The last reader unblocks the writer.
		// 说明所有的reader已经释放锁完毕，可以唤醒writer了
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

// 加锁
func (rw *RWMutex) Lock() {

	// First, resolve competition with other writers.
	// 首先，处理和其他writer之间的竞争
	rw.w.Lock()

	// Announce to readers there is a pending writer.

	// 将readerCount 进行反转，也就是设置成负数，用于告诉其他reader，有writer正在请求锁
	// 计算还有几个reader执行完成后，才能获得锁
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders

	// Wait for active readers.
	if r != 0 && rw.readerWait.Add(r) != 0 {
		// 等待数量 ！=0，说明有reader正在执行，使用信号量阻塞自己
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}

}

func (rw *RWMutex) Unlock() {

	// Announce to readers there is no active writer.

	// 反转readerCount为正数
	r := rw.readerCount.Add(rwmutexMaxReaders)

	if r >= rwmutexMaxReaders {
		race.Enable()
		fatal("sync: Unlock of unlocked RWMutex")
	}
	// Unblock blocked readers, if any.
	// 唤醒所有的reader
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	
	// 释放锁，允许其他writer获得锁
	rw.w.Unlock()

}
```