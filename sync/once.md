```go
type Once struct {
	done atomic.Uint32 // 0表示未初始化 1表示已初始化
	m    Mutex // 加锁，用来阻塞多个初始化流程
}

func (o *Once) Do(f func()) {

    // == 0 则表示可以初始化
	if o.done.Load() == 0 {
		o.doSlow(f)
	}
    
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock() // 函数退出，解锁

    // 获得锁后二次检查
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
        // 初始化函数执行完毕后，才会存储 done = 1
	}
}

```