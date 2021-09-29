```go
// 构建 next 数组
// next[i] = j 表示i位置往前j长度的字符串(不含) == 头部前j个字符
func buildNext(s string) {
	l := len(s)
	next := make([]int, l+1)
	j := 0
	for i := 1; i < l; i++ {
		for s[i] != s[j] && j > 0 {
			j = next[j]
		}
		if s[i] == s[j] {
			j++
			next[i+1] = j
		}
	}
}
```



#### 例题

[686. Repeated String Match](https://leetcode.com/problems/repeated-string-match/)

