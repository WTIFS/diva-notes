##### net/http 路由的实现

```go
type ServeMux struct {
	mu    sync.RWMutex         // 读写锁，保障哈希表的异步安全
	m     map[string]muxEntry  // 路由器真身，其实是原生golang hash map
	es    []muxEntry           // 路由器真身2号，用来实现前缀匹配.
	hosts bool                 // whether any patterns contain hostnames
}

type muxEntry struct {
    explicit bool
    h        Handler
    pattern  string
}
```

可以看到，里面的路由是用一个 `map` 和 路径切片 实现的。匹配路由时先查找 `map` 里有没一样的，没有的话再遍历 `map`，找前缀能匹配上的。

```go
// Most-specific (longest) pattern wins.
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
   
	v, ok := mux.m[path]  // 直接从hash map中查entry是否存在
	if ok {
		return v.h, v.pattern
	}

	for _, e := range mux.es { // 找前缀
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```





##### gin 路由的实现

`gin` 路由是通过前缀树实现的

```go
type methodTree struct {
	method string   // http方法， 每种方法存于一颗radix tree 实例
	root   *node    // 树的根节点
}

type node struct {
	path      string         // 到该节点为止的path
	children  []*node        // 子树
	handlers  HandlersChain  // 处理该url的handlers数组
}
```

