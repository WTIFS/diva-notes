##### net/http 路由的实现

是通过一个 `map` 和一个 切片实现的。注册一个路由时，会同时写入 `map` 和切片。查找路由时，先通过 `map` 尽快快速查找，`map` 里如果没有，则遍历切片，通过前缀查找。

```go
type ServeMux struct {
	mu    sync.RWMutex         // 读写锁，保障哈希表的异步安全
	m     map[string]muxEntry  // 路由器真身，其实是原生golang hash map
	es    []muxEntry           // 路由器真身2号，用来实现前缀匹配
	hosts bool                 // whether any patterns contain hostnames
}

type muxEntry struct {
    explicit bool
    h        Handler
    pattern  string
}
```

可以看到，里面的路由是用一个 `map` 和 路径切片 实现的。匹配路由时先查找 `map` 里有没一样的，没有的话再遍历切片，检查前缀，寻找第一个匹配上的。

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
   
	v, ok := mux.m[path]  // 直接从hash map中查entry是否存在
	if ok {
		return v.h, v.pattern
	}

	for _, e := range mux.es { // 遍历切片，找前缀
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern // 找到就返回
		}
	}
	return nil, ""
}
```



##### gorilla/mux

直接顺序遍历路由切片，使用正则匹配，返回第一个匹配上的。因此如果有多个相同开头的路由，应该尽量把精确的往前放

```go
func (r *Router) Match(req *http.Request, match *RouteMatch) bool {
	for _, route := range r.routes {
		if route.Match(req, match) {
            return true // 找到就返回
        }
    }
}
```



##### gin 路由的实现

`gin` 路由是通过前缀树实现的

```go
type methodTree struct {
	method string   // http方法， 每种方法存一颗 radix tree 实例
	root   *node    // 树的根节点
}

type node struct {
	path      string         // 到该节点为止的path
	children  []*node        // 子树
	handlers  HandlersChain  // 处理该url的handlers数组
  ...
}

// 注册路由
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
	// gin里是每个方法一棵树，如 GET/POST/PUT 分别一棵
  root := engine.trees.get(method)
  if root == nil {
    root = new(node)
    root.fullPath = "/"
    engine.trees = append(engine.trees, methodTree{method: method, root: root})
  }
  root.addRoute(path, handlers)
}
```

