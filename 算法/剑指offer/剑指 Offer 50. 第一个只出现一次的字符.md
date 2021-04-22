在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。 s 只包含小写字母。

```
s = "abaccdeff"
返回 "b"

s = "" 
返回 " "
```

```go
func firstUniqChar(s string) byte {
    hashtable := make(map[byte]int)
    for i := 0; i < len(s); i++ {
        hashtable[s[i]]++
    }
    for i := 0; i < len(s); i++ {
        if hashtable[s[i]] == 1 {
            return s[i]
        }
    }
    return ' '
}
```

