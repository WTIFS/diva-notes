并查集，用于判断两个点是否有相同的根节点。常用于图的连通性、根节点判断

```go
type UnionFind struct {
	Parents []int // 数组，记录每个点的根节点
	Count   int   // 根节点的数量（即数的数量，或者说不连通图的数量）
}

func (self *UnionFind) Init(n int) {
	self.Parents = make([]int, 0, n)
	self.Count = n
	for i := 0; i < n; i++ {
		self.Parents = append(self.Parents, i) // 初始化每个点的根节点为自己
	}
}

// 递归找u的根节点，并更新Parents数组
func (self *UnionFind) Find(u int) int {
    // 递归写法
	parent := self.Parents[u]
	if parent == u {
		return parent
	}
	root := self.Find(parent)
	self.Parents[u] = root
	return root
    
    // 非递归写法，效率更高
    root := self.Parents[u]
	for root != self.Parents[root] {
		root = self.Parents[root]
	}
	self.Parents[u] = root
	return root
}

// 连通u和v，将v的根节点指向u的根节点
func (self *UnionFind) Union(u, v int) bool {
	root1 := self.Find(u)
	root2 := self.Find(v)
	if root1 == root2 {
		return false
	}
	self.Parents[root2] = root1 // 将v的根节点指向u的根节点
	self.Count--                // 两个点连上了，不连通图的数量-1
	return true
}
```

