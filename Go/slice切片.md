##### slice结构

```go
struct {
	data unsafe.Pointer //指向数组中某一元素的指针
	len  int            //切片长度
	cap  int            //切片容量
}
```

因此可以通过 `unsafe.Pointer` 操纵地址来获取切片的 `data`、`len `和 `cap` 字段

```go
a := []int{3, 4}
arrayAddr := uintptr(unsafe.Pointer(&b))    // 切片起始地址
lenAddr := arrayAddr + unsafe.Sizeof(1)     // 长度地址
capAddr := arrayAddr + unsafe.Sizeof(1)*2   // 容量地址
println(*((*int)(unsafe.Pointer(lenAddr)))) // len = 2
println(*((*int)(unsafe.Pointer(capAddr)))) // cap = 2
```



##### 数组和切片的区别

1. 切片底层是数组，是对数组的封装，是截取了数组的一部分，所以叫切片。多个切片可以使用同一个底层数组，各切各的
2. 数组长度固定，声明对象时需指定长度；切片是变长的，可以自动扩容
3. 数组是值类型，可以比较，传参是值拷贝；切片是引用类型



##### nil 切片与空切片的区别

`nil` 切片没做初始化，没有分配内存

空切片只是底层数组没有初始化，其结构体本身、 `len`、`capacity` 字段都是有值的



##### append方法的实现

`append` 目前的源码是在编译期间生成的，在源码中并不能找到。但可以看早期版本的实现，逻辑是差不多的。下面是第一个版本的 `appendInt` 函数，专门用于处理 `[]int` 类型的切片

```go
func appendInt(x []int, y int) []int {
    var z []int
    zlen := len(x) + 1 // 先计算新长度
    if zlen <= cap(x) {
        z = x[:zlen] // 新长度 <= 容量，则修改切片长度
    } else { // 容量不够需扩容
        zcap := zlen
        if zcap < 2*len(x) {
            zcap = 2 * len(x)
        }
        z = make([]int, zlen, zcap) // 构建新切片，长度为新长度
        copy(z, x) // 复制旧切片数据至新切片
    }
    z[len(x)] = y // 赋值
    return z
}
```



##### 切片的扩容

切片插入的元素数量大于切片原长度时，直接扩容为插入元素后的长度。否则：

切片长度小于 `1024` 时，扩容为原容量的 `2` 倍，否则为 `1.25` 倍

比如 `append([1, 2], 3, 4, 5)`，翻倍后容量为 `4`，但原有 `2` 个元素加上新的 `3` 个元素后为 `5`，因此扩容后容量应为 `5`

不过之后还会进行内存对齐，实际最后的 `cap` 为 `6`

```go
// runtime/slice.go
// 参数：元素类型、旧切片、期望新切片的长度
func growslice(et *_type, old slice, cap int) slice {
	newcap := old.cap
	doublecap := newcap + newcap // 两倍
	if cap > doublecap { // 如果新增元素后长度 > 原两倍，直接用新增后的长度
		newcap = cap
	} else {
		if old.len < 1024 { // 小于 1024 翻倍
			newcap = doublecap 
		} else {
			for 0 < newcap && newcap < cap { // 不断翻 1.25 倍
				newcap += newcap / 4
			}
		}
	}
    
  switch {
    case et.size == 1: // 根据元素大小计算需要申请的空间
    capmem = roundupsize(uintptr(newcap)) // 内存对齐
    newcap = int(capmem)
  }
  
  // 复制旧数组元素到新数组
  p = mallocgc(capmem, et, true)
  memmove(p, old.array, lenmem)
  return slice{p, old.len, newcap}
}

// runtime/msize.go
// 内存对齐，获取最接近的 spanClass 的大小
// Returns size of the memory block that mallocgc will allocate if you ask for the size.
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
			return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
		} else {
			return uintptr(class_to_size[size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return alignUp(size, _PageSize)
}

```



##### 并发问题

切片并非并发安全的。不加锁并发 append 时会产生数据覆盖、数据缺少、零值数据的情况。

前两种问题比较好理解，零值是因为追加数据时，是先 1. 修改切片长度、2. 赋新值两步完成的。那么在刚完成 1未完成 2时，切片末尾元素便是零值。此时另一个协程若再追加数据，刚好触发扩容，便会导致零值元素被复制至扩容后的切片。



#### 参考

[Go语言圣经 - 4.2 Slice](http://books.studygolang.com/gopl-zh/ch4/ch4-02.html)
