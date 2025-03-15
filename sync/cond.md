```go
// Cond 常用于等待/通知的场景。
// Singal 唤醒一个等待队列中的goroutine。
// Broadcase 唤醒所有等待队列中的goroutine
// Wait阻塞自己，并且等待被唤醒。
type Cond struct {
	noCopy noCopy // 使用后不可被复制

	L Locker // 持有的锁

	notify  notifyList // 等待列表
	checker copyChecker
}

// NewCond returns a new Cond with Locker l.
// 创建Cond对象，需要指定一个锁
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}

// Wait atomically unlocks c.L and suspends execution
// of the calling goroutine. After later resuming execution,
// Wait locks c.L before returning. Unlike in other systems,
// Wait cannot return unless awoken by Broadcast or Signal.
//
// Because c.L is not locked while Wait is waiting, the caller
// typically cannot assume that the condition is true when
// Wait returns. Instead, the caller should Wait in a loop:
//
//	c.L.Lock()
//	for !condition() {
//	    c.Wait()
//	}
//	... make use of condition ...
//	c.L.Unlock()

// 调用Wait之前，需要持有锁，才能将自己加入等待队列。
// 被阻塞的goroutine在阻塞期间是不会持有锁的，一旦苏醒后，会立即获得锁。
// Wait 方法应该放到for循环中重复检查，直至条件满足。
func (c *Cond) Wait() {
	// c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}

// Signal wakes one goroutine waiting on c, if there is any.
//
// It is allowed but not required for the caller to hold c.L
// during the call.
//
// Signal() does not affect goroutine scheduling priority; if other goroutines
// are attempting to lock c.L, they may be awoken before a "waiting" goroutine.

// 不必在调用Signal的时候持有 c.L的锁
// Signal 唤醒一个等待队列中的goroutine。
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

// Broadcast wakes all goroutines waiting on c.
//
// It is allowed but not required for the caller to hold c.L
// during the call.

// Broadcast 唤醒所有等待的goroutine
// 不必在调用 Broadcast 的时候持有 c.L的锁
func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}

// noCopy may be added to structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
//
// Note that it must not be embedded, due to the Lock and Unlock methods.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

```