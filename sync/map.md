```go
// sync.Map 适用于两种情况
// 1. 写入一次，但是访问多次的情况。
// 2. 多个goroutine访问的key不相交的情况。
// 总的来说就是。《读多写少的情况》
type Map struct {
	// 互斥锁
	mu Mutex

	// read 字段，可以无锁并发访问
	// 修改read字段内的m，无须持有锁
	// 如果要修改这个字段，则必须要持有mu锁
	read atomic.Pointer[readOnly]

	// 包含需要加锁才能访问的entry
	dirty map[any]*entry

	// 统计未从read中读取，转而向dirty读取的次数
	misses int
}

// 只读类型
type readOnly struct {
	m       map[any]*entry
	amended bool // true if the dirty map contains some key not in m.
}

// An entry is a slot in the map corresponding to a particular key.
type entry struct {

    // p == nil 说明是软删除，其entry 还没有被删除
    // p == expunged 说明这个key已经被硬删除了，需要重新构建一个entry
    // 存储具体key所对应value的地方
	p atomic.Pointer[any]
}

// 尝试使用i的值替换entry的p指针
func (e *entry) trySwap(i *any) (*any, bool) {
	for {
		// 加载出entry的p指针
		p := e.p.Load()

		// 指针状态表示当前的entry已经被标记删除了，
		if p == expunged {
			return nil, false
		}

		// 尝试将当前的指针i cas赋值到p上面
		if e.p.CompareAndSwap(p, i) {
			return p, true
		}
	}
}

// 存一个key，value，或者是更新
func (m *Map) Store(key, value any) {
	_, _ = m.Swap(key, value)
}

// previous 之前存在的值
// loaded 值是否被找到
func (m *Map) Swap(key, value any) (previous any, loaded bool) {


	// 路径总结
	// 1. 运气好，存在于read中，直接cas更新
	// 2. 获得锁，双检查，存在于read中，但是被标记为删除了，尝试恢复
	// 3. 存在于dirty中，直接修改p的地址
	// 4. 创建一个新的dirty，将read中存在的其他正常entry拷贝到dirty中，并且将这个key加入到dirty中

    // 获得只读map
	read := m.loadReadOnly()

	// 首先检查key是否存在于read中
	if e, ok := read.m[key]; ok {

        // 当前key存在于只读map中，继续检查，
		if v, ok := e.trySwap(&value); ok {

			// 成功找到值，检查是否存在旧值
			if v == nil {
				return nil, false
			}
			return *v, true
		}
	}

    // 只读map中不存在，需要继续检查dirty了
	// 加锁
	m.mu.Lock()
	read = m.loadReadOnly()

	// 双检查
	if e, ok := read.m[key]; ok {
		// 继续在read中找到了这个key

		// 检查它的entry是否被标记为删除，如果标记为删除了，就尝试恢复
		if e.unexpungeLocked() {
			// The entry was previously expunged, which implies that there is a
			// non-nil dirty map and this entry is not in it.

			// 恢复成功，将e加入到dirty中
			m.dirty[key] = e
		}
		if v := e.swapLocked(&value); v != nil {

			// v != nil 说明之前有值存在，加载值

			loaded = true
			previous = *v
		}
	} else if e, ok := m.dirty[key]; ok {

		// 进入到这儿，read中没有存在，但是dirty中存在

		// 尝试替换
		if v := e.swapLocked(&value); v != nil {
			loaded = true
			previous = *v
		}
	} else {

		// 既不存在于read中
		// 也不存在于dirty中

		// read.amended == false 说明没有key在dirty中存在，说明这时候dirty还没有被创建，需要创建一个
		if !read.amended {
			// 创建一个新的dirty
			// 将read中的，正常的entity加入到dirty中
			// 并且设置 amended = true，说明有key不存在于read中，但存在于dirty中

			m.dirtyLocked()
			m.read.Store(&readOnly{m: read.m, amended: true})
		}
		// 将当前dirty的key进行赋值
		m.dirty[key] = newEntry(value)
	}
	m.mu.Unlock()
	return previous, loaded
}

func (e *entry) unexpungeLocked() (wasExpunged bool) {
	return e.p.CompareAndSwap(expunged, nil)
}

func (m *Map) dirtyLocked() {

	// 存在dirty，直接返回
	if m.dirty != nil {
		return
	}

	read := m.loadReadOnly()

	// 创建一个新的dirty，长度和read的m一致
	m.dirty = make(map[any]*entry, len(read.m))

	// 遍历read中的map
	for k, e := range read.m {

		// k 的类型是 entity. k结构体的field p 有三种情况

		// nil -> 会转换为expunged，表示当前元素现在只存在于read中
		// expunged -> 会被忽略，也就是丢弃掉了
		// 正常指针 -> 会被加入到dirty中，read和dirty中都会存在了。

		// 所以一个元素entity被真正删除的时机在于：
		// 1. dirty 转换为read的时候，expunged会被忽略，导致其不会被引用，会被回收
		// 2. 如果key存在于dirty中，进行delete删除的时候。

		// 只有非 expunged 的元素 才能加入到dirty中
		// tryExpungeLocked 也会将 元素的P是nil 的值，转换为 expunged。
		// 表示这个key，当前只存在于read中，但是不存在于dirty中
		if !e.tryExpungeLocked() {
			m.dirty[k] = e
		}
	}
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
	p := e.p.Load()
	for p == nil {
		if e.p.CompareAndSwap(nil, expunged) {
			return true
		}
		p = e.p.Load()
	}
	return p == expunged
}


// Load方法用于加载key所对应的值，如果找到，就返回，否则就返回nil.
// ok 表示是否存在这个key的标志位
func (m *Map) Load(key any) (value any, ok bool) {

    // 不加锁的read map首先读取
	read := m.loadReadOnly()
	e, ok := read.m[key]

	if !ok && read.amended {

		// 不存在于read中，并且 amended=true，表示有key存在于dirty中

        // 当前goroutine执行到这儿的时候，可能其他gorutine会把read进行修改

        // 未读取到，尝试加锁
		m.mu.Lock()

        // 二次检查
		read = m.loadReadOnly()
		e, ok = read.m[key]

		if !ok && read.amended { // 仍然不存在于read中，但需要检查dirty.

            // 直接返回dirty中的信息，无论是否有这个key
			e, ok = m.dirty[key]
            // miss + 1
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load() //返回read的值，这个e有可能是从dirty中读取的，也有可能是从read中读取的
}

func (e *entry) load() (value any, ok bool) {
	p := e.p.Load()
	if p == nil || p == expunged {
		return nil, false
	}
	return *p, true
}

func (m *Map) missLocked() {
	// miss次数增加
	m.misses++
	// miss次数小于dirty的长度，则退出
	if m.misses < len(m.dirty) {
		return
	}

	// 如果miss次数大于等于dirty的长度了，就需要转换
	// 将dirty的指针直接赋值给read
	m.read.Store(&readOnly{m: m.dirty})
	// 给dirty设置nil
	m.dirty = nil
	// 重置miss次数
	m.misses = 0
}


// 从entry中获得key对应的值
func (e *entry) load() (value any, ok bool) {
	p := e.p.Load()
	if p == nil || p == expunged {
		return nil, false
	}
	return *p, true
}


// Delete deletes the value for a key.
func (m *Map) Delete(key any) {
	m.LoadAndDelete(key)
}

// 删除一个key，并且返回其之前的值，
// 如果之前存在值，则 loaded = true,否则就是false
func (m *Map) LoadAndDelete(key any) (value any, loaded bool) {
	read := m.loadReadOnly() // 加载只读map
	e, ok := read.m[key]
	if !ok && read.amended {// 不存在于只读map,但是存在于dirty中
        // 加锁
		m.mu.Lock()
        // 双检查
        // 双检查的目的在于，当前goroutine 进入if方法，准备加锁，被其他goroutine正好修改了，
		read = m.loadReadOnly()
		e, ok = read.m[key]
		if !ok && read.amended { // 仍然不存在于只读map中，

            // 访问dirty，查询旧值
			e, ok = m.dirty[key]

            // 直接从dirty中剔除这个key
			delete(m.dirty, key)
            // miss + 1
			m.missLocked()
		}
        // 解锁
		m.mu.Unlock()
	}
    // 存在这个entry
	if ok {
        //
		return e.delete()
	}
	return nil, false
}


// entry 进行删除操作
func (e *entry) delete() (value any, ok bool) {
	for {
		p := e.p.Load()
		if p == nil || p == expunged {
			return nil, false
		}
        // 当前entry存在旧值，尝试cas设置p==nil
		if e.p.CompareAndSwap(p, nil) {
			return *p, true
		}
	}
}
```