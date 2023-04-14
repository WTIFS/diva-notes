```go
            i
比如在 a b a b a d 中
查找： a b a c
            j
```

KMP的重点就是 发现 b 和 c 不匹配时，应该知道指针下一步要移动到哪？

在上面的例子中，字符串1指针 `i` 不动，字符串2指针 `j` 回到 b 那里。因为 c 前面的 a 和字符串头部的 a 一样，而 a 在之前的比较中已经验证过可以和字符串1匹配了，因此不需要再比较这个 a 了

```
      i
a b a b a d
    a b a c
      j
```

由此可以观察到这样的性质：移动到的地方的 k个字符和最前面 k个字符是一样的

`buildNext` 就是用来标记每个位置上有多长的字符串和开头一样

```go
next[i] = k  //表示i位置往前k长度的字符串(不含) == 头部前 k 个字符

B:      b c b c d
next:   0 0 0 1 2 0
```





```go
// 构建 next 数组
// next[i] = k
// 表示i位置往前k长度的字符串(不含) == 头部前k个字符
// B:      b c b c d
// next:   0 0 0 1 2 0
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

