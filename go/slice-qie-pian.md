##### 数组和切片的区别

1. 切片底层是数组，是对数组的封装，是截取了数组的一部分，所以叫切片。多个切片可以使用同一个底层数组，各切各的
2. 数组长度固定，声明对象时需指定长度；切片是变长的，可以自动扩容
3. 数组是值类型，可以比较，传参是值拷贝；切片是引用类型



##### nil 切片与空切片的区别

`nil` 切片没做初始化，没有分配内存

空切片只是底层数组没有初始化，其结构体本身、 `len`、`capacity` 字段都是有值的



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
	case et.size == 1:
		capmem = roundupsize(uintptr(newcap)) // 内存对齐
		newcap = int(capmem)
    }
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





#### 参考

[盼盼 - Go 切片只需这一篇](https://mp.weixin.qq.com/s/sSAsTXuxIOc9b9N1lclCxQ)

