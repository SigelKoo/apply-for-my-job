给你一个字符串 `s`，找到 `s` 中最长的回文子串。

```
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
```

```
输入：s = "cbbd"
输出："bb"
```

```
输入：s = "a"
输出："a"
```

```
输入：s = "ac"
输出："a"
```

```go
func longestPalindrome(s string) string {
    res := s[: 1]
    for i := 0; i < len(s); i++ {
        s1 := palindrome(s, i, i)
        s2 := palindrome(s, i, i + 1)
        if len(s1) > len(res) {
            res = s1
        }
        if len(s2) > len(res) {
            res = s2
        }
    }
    return res
}

func palindrome(s string, l, r int) string {
    for l >= 0 && r <= len(s) - 1 && s[l] == s[r] {
        l--
        r++
    }
    return s[l + 1 : r]
}
```

