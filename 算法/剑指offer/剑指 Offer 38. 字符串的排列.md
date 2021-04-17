#  字符串的排列

输入一个字符串，打印出该字符串中字符的所有排列。

你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。

```
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

```go
func permutation(s string) []string {
	res := make(map[string]bool, 0)
	ss := []rune(s)
	dfs38(ss, res, []int{}, len(ss))
	list := []string{}
	for k, _ := range res {
		list = append(list, k)
	}
	return list
}

func dfs38(ss []rune, res map[string]bool, used []int, n int) {
	if len(used) == n {
		r := []rune{}
		for _, v := range used {
			r = append(r, ss[v])
		}
		res[string(r)] = true
		return
	}
	for i := range ss {
		if listContain(used, i) {
			continue
		}
		used = append(used, i)
		dfs38(ss, res, used, n)
		used = used[:len(used) - 1]
	}
}

func listContain(used []int, a int) bool {
	for _, v := range used {
		if a == v {
			return true
		}
	}
	return false
}
```