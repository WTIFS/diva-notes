# 概述
其实底层还是用的 `map`, 就是对 `map` 做了封装。主要思想是冗余的用了2个 `map`，`read` 和 `dirty`, 读写分离。
读写的时候先找 `read`, 不加锁, 写的时候使用 `CAS`; 之后再找 `dirty`。读写 `dirty` 时才加锁。



# 实现

## 数据结构

在 src/sync/map.go 中

```go
type Map struct {
    // 当涉及到脏数据(dirty)操作时候，需要使用这个锁
    mu Mutex
    
    // read是一个只读数据结构，包含一个map结构，
    // 读不需要加锁，只需要通过 atomic 加载最新的指针即可
    read atomic.Value // readOnly
    
    // dirty 包含部分map的键值对，如果操作需要mutex获取锁
    // 最后dirty中的元素会被全部提升到read里的map去
    dirty map[interface{}]*entry
    
    // misses是一个计数器，用于记录read中没有的数据而在dirty中有的数据的数量。
    // 也就是说如果read不包含这个数据，会从dirty中读取，并misses+1
    // 当misses的数量等于dirty的长度，就会将dirty中的数据迁移到read中
    misses int
}
```

**read的数据结构 readOnly：**
```go
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
    // m包含所有只读数据，不会进行任何的数据增加和删除操作 
    // 但是可以修改entry的指针因为这个不会导致map的元素移动
    m       map[interface{}]*entry
    
    // 标志位，如果为true则表明当前read只读map的数据不完整，dirty map中包含部分数据
    amended bool // true if the dirty map contains some key not in m.
}
```

**entry**
```go
type entry struct {
    p unsafe.Pointer // *interface{}
}
```
p有三种值：
- nil: entry已被删除了，并且m.dirty为nil
- expunged: entry已被删除了，并且m.dirty不为nil，但这个entry在于m.dirty中
- 其它：entry是一个正常的值



## 查找

```go
// src/sync/map.go

// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 首先从只读ready的map中查找，这时不需要加锁
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // 如果没有找到，并且read.amended为true，说明dirty中有新数据，从dirty中查找，开始加锁了
    if !ok && read.amended {
        m.mu.Lock() // 加锁
        
       // 又在 readonly 中检查一遍，因为在加锁的时候 dirty 的数据可能已经迁移到了read中
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        
        // read 还没有找到，并且dirty中有数据
        if !ok && read.amended {
            e, ok = m.dirty[key] //从 dirty 中查找数据
            
            // 不管m.dirty中存不存在，都在missLocked() 中将misses + 1。misses = len(m.dirty)时就会把m.dirty中的数据迁移到m.read中
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

 // 即使数据不在read中，经过几次miss后， m.dirty中的数据也会迁移到m.read中，这时又可以从read中查找。
func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) { //misses次数小于 dirty的长度，就不迁移数据，直接返回
        return
    }
    m.read.Store(readOnly{m: m.dirty}) //开始迁移数据
    m.dirty = nil   //迁移完dirty就赋值为nil
    m.misses = 0    //迁移完 misses归0
}
```



## 新增和更新

对 `read` 中已有的key，通过 `CAS` 来更新值；否则加锁并写入 `dirty`

```go
// src/sync/map.go

// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
   // 直接在read中查找值，找到了，就尝试 tryStore 更新值, tryStore 里使用 for 循环 + CAS 的方式进行更新
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    // m.read 中不存在
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() { // 未被标记成删除
            m.dirty[key] = e     // 加入到dirty里
        }
        e.storeLocked(&value) // 设置值
    } else if e, ok := m.dirty[key]; ok { // 存在于 dirty 中，直接更新
        e.storeLocked(&value)
    } else { // 新的值
        if !read.amended { // m.dirty 中没有新数据，增加到 m.dirty 中
            m.dirtyLocked() // 从 m.read中复制未删除的数据
            m.read.Store(readOnly{m: read.m, amended: true}) // 将read.amended字段标记为true，下次查找会启用dirty查找
        }
        m.dirty[key] = newEntry(value) //将这个entry加入到m.dirty中
    }
    m.mu.Unlock()
}
```



## 删除

和新增差不多，只不过不设置值了，而是调用底层 `map` 的 `delete` 方法




# 总结
1. 空间换时间：通过冗余的两个数据结构(read、dirty)，减少加锁对性能的影响。
2. 使用只读数据(read)，避免读写冲突。
3. 动态调整，miss次数多了之后，将dirty数据迁移到read中。
4. double-checking。
5. 延迟删除。 删除一个键值只是打标记 (go的map都是这样，这个不是sync.Map特有的)。只有在迁移dirty数据的时候才清理删除的数据。
6. 优先从read读取、更新、删除，因为对read的读取不需要锁。
7. 适合读多写少的场景。写多的情况下，仍然会频繁的加锁，且是全局锁。写多的场景，可以借鉴 `java 1.7 concurrent hashmap` 的实现方法，使用分段锁，降低锁粒度。也有人这么做了，如[concurrent-map](https://github.com/orcaman/concurrent-map)