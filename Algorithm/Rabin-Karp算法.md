和 KMP 一样用作在字符串中快速查找一个子字符串是否存在，不过比 KMP 好理解。



其原理是：

如果我们已知字符串 "abcd" 中 "abc" 的哈希值，那么可以用 O(1) 的时间算出右移1格后字符串 "bcd" 的哈希值



具体说明：

假设有字符串 a，设其 `hash(a) = x`，设进制为26

对字符串 ab，其 `hash(ab) = hash(a)*26 + hash(b)`

对字符串 abc，其 `hash(abc) = hash(ab)*26 + hash(c) = hash(a)*26² + hash(b)*26 + hash(c)`

则对字符串 bcd：

```
hash(bcd) = hash(bc)*26 + hash(d)
          = (hash(abc) - hash(a)的部分)*26 + hash(d)
          = (hash(abc) - hash(a)*26²)*26 + hash(d)
```

也就是说，`hash(bcd)` 可以根据 `hash(abc)` 的结果用 O(1) 的时间算出来，而不用 O(N) 遍历 "bcd" 三个字母算，这就是该算法高效的原理

不过这样算出的 hash 有概率冲突，所以 hash 相同的情况下需要检查下原始字符串





```go
func rabinKarp(haystack string, needle string) int {
    if len(haystack) < len(needle) {
        return -1
    }

    base := 26 // 进制，假设输入只包含26个字母，这里就可以使用26进制
    mod := int(1e5) // 为防止 int 溢出取模
    l := len(needle)

    targetHash := 0 // 要查找的 hash
    curHash := 0    // 当前比对字符串的 hash
    
    // shift 用于算最高位有几次方，用于算滚动 hash 时扣减最高位。比如 hash(abc) = a*26² + b*26 + c，则 shift = 26² 
    shift := 1 

    // 计算初始 hash
    for i := range needle {
        if i > 0 {
            shift = shift * base % mod
        }
        targetHash = (targetHash*base + byteToInt(needle[i])) % mod
        curHash = (curHash*base + byteToInt(haystack[i])) % mod
    }

    // 这里第一步 i 是 exclude 的，因此条件为 <=
    for i := len(needle); i <= len(haystack); i++ {
        // 1. 比对 hash 和原始字符串
        if curHash == targetHash && haystack[i-l:i] == needle {
            return i - len(needle)
        }
        // 2. 计算滚动 hash
        if i < len(haystack) {
            curHash = ((curHash-(byteToInt(haystack[i-l])*shift))*base + byteToInt(haystack[i])) % mod
            if curHash < 0 {
                curHash += mod
            }
        }
    }

    return -1
}

func byteToInt(b byte) int {
    return int(b - 'a')
}
```
