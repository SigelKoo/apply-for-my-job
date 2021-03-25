#  字符串的排列

输入一个字符串，打印出该字符串中字符的所有排列。

你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。

```
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

```go
func permutation(s string) []string {
    return getString(s)
}

func getString(s string) []string {
    var ret []string
    if len(s) == 0 {
        return ret
    }
    if len(s) == 1 {
        ret = append(ret, s)
        return ret
    }
    maps := make(map[string]bool)
    for i, val := range s {
        if maps[string(val)] == false {
            maps[string(val)] = true
            var tmp string
            tmp = s[0:i] + s[i+1:]
            for _, v := range getString(tmp) {
                ret = append(ret, string(val) + v)
            }
        }
    }
    return ret
}
```