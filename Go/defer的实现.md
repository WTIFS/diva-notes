# 基本原理

`go` 里的 `defer` 可以将函数延迟至返回前再执行。相比手动在 `return` 前执行函数来说，一是可以使开启和结束的代码放在一起，清晰的同时避免忘写结束代码；另外即便 `panic` 时函数也可以继续执行，这是手动写做不到的。

其基本原理是，编译器会对代码做内联优化， 把 `defer` 部分的函数插入到返回之前



## go1.13 之前

编译期在 `defer` 位置生成一个 `runtime.deferproc` 调用，并在函数退出时生成一个 `runtime.deferreturn` 调用

- `deferproc` 负责构造一个用来保存  **函数地址** 以及 **函数参数** 的 `_defer` 结构体对象，并把该对象入栈，即插入 `G` 结构体对象的 `_defer` 链表表头
- `deferreturn` 负责从栈里读取函数及参数，并依次执行

比如如下代码：

```go
func run() {
    defer foo()
    defer bar()

    fmt.Println("hello")
}
```

编译器会生成类似如下的代码：

```go
runtime.deferproc(foo) // generated for line 1
runtime.deferproc(bar) // generated for line 2

fmt.Println("hello")

runtime.deferreturn() // generated for line 5
```

这使得使用 `defer` 时增加了 `runtime` 函数调用开销。

另外 由于使用了先进后出的栈管理着一个函数中的多个 `defer` 调用，因此 `defer` 越多，开销越大。



## go1.13

在 `go1.13` 之前所有 `defer` 都是在堆上分配。`go1.13` 版本新加入 `deferprocStack` 实现了在栈上分配 `defer`。

相比堆上分配，栈上分配在函数返回后 `_defer` 便得到释放，省去了内存分配时产生的性能开销，只需适当维护 `_defer` 的链表即可。按官方文档的说法，这样做提升了约 30% 左右的性能。

除了分配位置的不同，栈上分配和堆上分配并没有本质的不同。

> 另外，1.13 版本中并不是所有 `defer` 都能够在栈上分配。循环中的 `defer`，无论是显示的 `for` 循环，还是 `goto` 形成的隐式循环，都只能使用堆上分配。



## go1.14

`go 1.14` 版本加入了开放编码（open coded），该机制会把 `defer` 调用的代码直接内联到返回之前，省去了 `runtime` 的操作。

还拿上面那个例子举例，编译器将生成如下的代码：

```go
fmt.Println("hello")

bar() // generated for line 5
foo() // generated for line 5
```

>  但是 `defer` 并不是所有场景都能内联。比如 在一个循环次数可变的循环中 / `defer` 数量超 8 个 等场景下就没法内联。但我们大部分时候也不会在这些场景下使用 `defer`。



# 源码

##### _defer 结构体

```go
// src/runtime/runtime2.go
type _defer struct {
    siz       int32    // 参数和结果的大小
    heap      bool     // 是否是堆上分配
    openDefer bool     // 开放编码
    sp        uintptr  // 栈指针
    pc        uintptr  // 调用方的程序计数器 program counter
    fn        *funcval // 传入的函数
    _panic    *_panic  // panic that is running defer
    link      *_defer  // defer链表
}
```



##### deferproc

主要做的事情就是把 `defer` 后的函数、参数写入 `_defer` 结构体里，并将其插入到 `G` 的 `defer` 链表头部

```go
// src/runtime/panic.go
func deferproc(siz int32, fn *funcval) {  
  	sp := getcallersp()                                      // 获取栈指针
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn) // fn 函数后紧跟的就是参数列表
	callerpc := getcallerpc()
	
	d := newdefer(siz) 	// 新建一个 _defer
    d.link = gp._defer  // 把原链表挂在新 defer 后面
	gp._defer = d       // 把链表写入G里
  
    d.fn = fn
	d.pc = callerpc
  	d.sp = sp
    
    // 进行参数拷贝
    switch siz {
    // 如果defered函数的参数只有指针大小则直接通过赋值来拷贝参数
    case sys.PtrSize:
        // 将 argp 所对应的值 写入到 deferArgs 返回的地址中
        *(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
    default:
        // 如果参数大小不是指针大小，那么进行数据拷贝
        memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
    }
   	return0() 
}
```



##### deferreturn

```go
// src/runtime/panic.go
func deferreturn(arg0 uintptr) {
    gp := getg()
    d := gp._defer
    
    sp := getcallersp() // 确定 defer 的调用方是不是当前 deferreturn 的调用方
    if d.sp != sp {
      return
    }

    switch d.siz {
    case sys.PtrSize:
      	// 将 defer 保存的参数复制出来
     	  // arg0 实际上是 caller SP 栈顶地址值，所以这里实际上是将参数复制到 caller SP 栈顶地址值
      	*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
    default:
        // 如果参数大小不是 sys.PtrSize，那么进行数据拷贝
      	memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
    }
  
    gp._defer = d.link // 将 defer 从链表移出
    freedefer(d)       // 将 defer 对象放入到 defer 池中，后面可以复用

    fn := d.fn
    _ = fn.fn
  
    // 这里是实际执行函数的部分，jumpdefer 是汇编代码
    jmpdefer(fn, uintptr(unsafe.Pointer(&arg0))) // 传入需要执行的函数和参数
}
```





# panic

顺带一提，`panic` 的实现和 `defer` 息息相关，也写在这里。

先来看 `G` 的结构。`G` 的字段里分别有两个链表，储存 `panic` 和 `defer` 删除

```go
type g struct {
    _panic       *_panic   // panic链表
	_defer       *_defer   // defer链表
}
```

编译器将 `panic` 翻译成 `gopanic` 函数调用。它会将错误信息打包成 `_panic` 对象，并挂到 `G._panic` 链表的头部。然后遍历 `G._defer` 链表，检查是否有 `recover`。如被 `recover`，则终止遍历执行，跳转到正常的 `deferreturn` 环节；否则终止进程。





#### 参考

> [深入理解defer（下）defer实现机制](https://zhuanlan.zhihu.com/p/69455275)

