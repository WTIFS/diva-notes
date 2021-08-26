# 概述
其实底层还是用的 `map`，就是对 `map` 做了封装。主要思想是做分区，底层用了2个 `map`，`read` 和 `dirty`，将频繁变更的数据，和极少变更的数据分离。

极少变更的数据放到 `read` 里，读 `read` 无需加锁，更新 `read` 依靠 `CAS` 也不用加锁，性能极高。

新增的数据放到 `dirty` 里。这类数据的读写需要加锁。

读取数据时，先查 `read`，再查 `dirty`，当读 `read` cache miss 累计到一定次数时，则将数据从 `dirty` 移入 `read`



# 实现

## 数据结构

```go
// src/sync/map.go
type Map struct {
    // 当涉及到脏数据(dirty)操作时候，需要使用这个锁
    mu Mutex
    
    // read 字段指向下面的 readOnly 结构，里面有个 map
    // 读/更新不需要加锁，只需要通过 atomic 加载最新的指针即可
    read atomic.Value 
    
    // dirty 包含部分 map 的键值对，如果操作需要 mutex 获取锁
    // 新增键值时写入这个字段
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
    // m 包含所有只读数据，不会进行任何的数据增加和删除操作 
    // 但是可以修改 entry 的指针因为这个不会导致map的元素移动
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

先查找 `read` ，`read` 中没有再查找 `dirty` ，并增加 `cache misses` 计数

`cache misses` 数量累计至 `dirty` 的长度后，将 `dirty` 数据迁移至 `read` 中

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

如果 `read` 中已有 `key`，通过 `CAS` 来更新值；

新增 `key`，或者更新 `dirty` 中的数据，需要加锁并写入 `dirty`

```go
// src/sync/map.go

// Store sets the value for a key.
func (m *Map) Store(key, value interface{}) {
   // 直接在 read 中查找值，找到了，就尝试 tryStore 更新值, tryStore 里使用 for 循环 + CAS 的方式进行更新
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }
    
    // m.read 中不存在
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok { // double check，因为 lock 前 read 里可能又有了
        if e.unexpungeLocked() {  // 未被标记成删除
            m.dirty[key] = e      // 加入到dirty里
        }
        e.storeLocked(&value)             // 设置值
    } else if e, ok := m.dirty[key]; ok { // 存在于 dirty 中，直接更新
        e.storeLocked(&value)
    } else {                // read 和 dirty 中都没有，即插入新数据
        if !read.amended {  // m.dirty为空，即向 dirty 中第一次加载数据
            m.dirtyLocked() // 会将 read 里的所有数据加载到 dirty 中，并将 read 里的键值标记为 expunged
            m.read.Store(readOnly{m: read.m, amended: true}) // 将 read.amended 字段标记为true，下次查找会启用dirty查找
        }
        m.dirty[key] = newEntry(value) // 将这个entry加入到m.dirty中
    }
    m.mu.Unlock()
}
```



## 删除

和新增差不多，只不过不设置值了，而是调用底层 `map` 的 `delete` 方法




# 总结
1. 空间换时间：通过冗余的两个数据结构 (read、dirty)，减少加锁对性能的影响。
2. 使用只读数据 (read)，避免读写冲突。
3. 动态调整，miss 次数多了之后，将 dirty 数据迁移到read中。
4. double-checking。
5. 优先从 read 读取、更新、删除，因为对 read 的读取不需要锁。
7. 适合读多写少的场景。写多的情况下，仍然会频繁的加锁，且是全局锁。写多的场景，可以借鉴 `java 1.7 concurrent hashmap` 的实现方法，使用分段锁，降低锁粒度。也有人这么做了，如 [concurrent-map](https://github.com/orcaman/concurrent-map)





## 问题

##### read 里也是 map ，扩容时咋办？不需要加锁吗？

