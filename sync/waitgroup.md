```go
type WaitGroup struct {
	noCopy noCopy

	state atomic.Uint64 // high 32 bits are counter, low 32 bits are waiter count.
	sema  uint32
}
// add 方法添加一个delta到计数器上，这个delta可以是负数
// 如果计数器归零，则阻塞在这个waitgroup上面的所有goroutine，会被唤醒
// 如果计数器小于0，则会panic
//
// Note that calls with a positive delta that occur when the counter is zero
// must happen before a Wait. Calls with a negative delta, or calls with a
// positive delta that start when the counter is greater than zero, may happen
// at any time.
// Typically this means the calls to Add should execute before the statement
// creating the goroutine or other event to be waited for.
// If a WaitGroup is reused to wait for several independent sets of events,
// new Add calls must happen after all previous Wait calls have returned.
// See the WaitGroup example.
func (wg *WaitGroup) Add(delta int) {

    // state字段，高32位作为计数器使用 低32位作为等待的数量
    // 添加delta 到 计数器上
	state := wg.state.Add(uint64(delta) << 32)

    // 获得当前的计数器值
	v := int32(state >> 32)

    // state(64位) 转换为32 位，会发生高位截取，所以这里w代表的就是 state的低32位
    // waiter counter
	w := uint32(state)

    // 计数器最多到0，不能是负数
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	// add的调用，要完全在wait之前，无如果 w!=0,则说明已经有等待者出现了。
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

    // 当前计数器>0 说明仍然有goroutine未执行完，继续等待
	// 或者 w=0 说明等待数量=0，就直接结束
	if v > 0 || w == 0 {
		return
	}

	// 执行到这儿的话，说明 v=0,计数器归零了，说明所有的goroutoine已经调用了Done，执行完毕
	// 并且 有等待者 w > 0. 那么需要进行唤醒。

    // 发现并发问题，发现state已经被其他goroutine改变，这是不被允许的
	if wg.state.Load() != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}

	// Reset waiters count to 0.
	// 重新设置w==0，说明没有等待者
	
	wg.state.Store(0)
	// 唤醒所有的等待者
	for ; w != 0; w-- {
		runtime_Semrelease(&wg.sema, false, 0)
	}
}
// Done 就很简单了，只是对计数器-1
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}


func (wg *WaitGroup) Wait() {

    // ignore race

	for {

        // 一直循环，获取state值
		state := wg.state.Load()

        // 计数器的值
		v := int32(state >> 32)
        // 等待者的数量
		w := uint32(state)

        // 计数器归零，不需要等待了
		if v == 0 {
			return
		}

		// 计数器 > 0 ，需要等待
        // 等待者数量 +1
        // 尝试设置这个值，多次尝试
		if wg.state.CompareAndSwap(state, state+1) {

            // 调用信号量，阻塞自己
			runtime_Semacquire(&wg.sema)
            // 阻塞完成，被唤醒

            // 醒来的话，说明其他goroutine 通过Done方法，将计数器归零了，
            // 有这样一句 wg.state.Store(0)
            // 会设置为0

            // 这里不等于0，则说明在Wait返回之前，又被再次使用了。
			if wg.state.Load() != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}

            // 退出循环
			return
		}
	}
}
```