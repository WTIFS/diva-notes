Tarjan 算法是基于深度优先搜索的算法，用于求解图的连通性问题。Tarjan 算法可以在线性时间内求出无向图的割点与桥，进一步地可以求解无向图的双连通分量；同时，也可以求解有向图的强连通分量、必经点与必经边。



## 无向图的割点与桥

##### 割点

若从图中删除节点 x 以及所有与 x 关联的边之后，图将被分成两个或两个以上的不相连的子图，那么称 x 为图的**割点**。



##### 桥

若从图中删除边 e 之后，图将分裂成两个不相连的子图，那么称 e 为图的 **桥** 或 **割边**。



##### 强连通分量

互相可达的点（如环），称为强连通分量。即除了桥以外的边都是强连通分量





## 算法

##### 时间戳

时间戳是用来标记图中每个节点在进行深度优先搜索时被访问的时间顺序，你可以理解成一个序号（这个序号由小到大），或者把图组成一个树，则时间戳即为节点的深度。用 `dfn[x]` 来表示。



##### 追溯值

追溯值用来表示从当前节点 `x` 作为搜索树的根节点出发，能够访问到的所有节点中，时间戳最小的值。用 `low[x]` 表示。

当无环时， `low[x] = dfn[x]`

有环时，`low[x] = 环里最小的 dfn[]`

这样通过比较 `low[x] 和 dfn[x]`，我们就可以判断是否存在环。把有环的都排除掉，剩下的就是割边了



```go
tarjan(u) {
    DFN[u] = Low[u] = ++Index                 // 为节点u设定次序编号和Low初值
    Stack.push(u)                             // 将节点u压入栈中
    for each (u, v) in E                      // 枚举每一条边
          if (v is not visted)                // 如果节点v未被访问过
                tarjan(v)                     // 继续向下找
                Low[u] = min(Low[u], Low[v])
          else if (v in S)                  // 如果节点v还在栈内
            	Low[u] = min(Low[u], DFN[v])
    if (DFN[u] == Low[u])             // 如果节点u是强连通分量的根
       repeat
           v = S.pop                  // 将v退栈，为该强连通分量中一个顶点
           print v
      until (u== v)
} 
```





### 例题

[1192. Critical Connections in a Network](https://leetcode.com/problems/critical-connections-in-a-network/)





#### 参考
[60 分钟搞定图论中的 Tarjan 算法](https://zhuanlan.zhihu.com/p/101923309)

