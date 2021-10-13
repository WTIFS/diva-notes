```go
// 构建 next 数组
// next[i] = j 表示i位置往前j长度的字符串(不含) == 头部前j个字符
// A:    a b c b c b      a b c b c b
// B:      b c b c d  =>        b c b c d
// next:   0 0 0 1 2
func buildNext(s string) []int {
	next := make([]int, len(s)+1)
	j := 0
	for i := 1; i < len(s); i++ {
		for s[i] != s[j] && j != next[j] {
			j = next[j]
		}
		if s[i] == s[j] {
			j++
			next[i+1] = j
		}
	}
	return next
}

// 判端 needle 在 heystack 中出现的下标
func strStr(haystack string, needle string) int {
	i, j := 0, 0
	m, n := len(haystack), len(needle)
	if n == 0 {
		return 0
	}

	next := buildNext(needle)
	for i < m && j < n {
		if haystack[i] == needle[j] {
			i++
			j++
		} else {
			if j > 0 {
				j = next[j]
			} else {
				i++
			}
		}
		if j == n {
			return i - j
		}
	}
	return -1
}
```



#### 例题

[28. Implement strStr](https://leetcode.com/problems/implement-strstr)

[686. Repeated String Match](https://leetcode.com/problems/repeated-string-match/)

