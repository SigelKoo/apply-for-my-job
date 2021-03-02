# 替换空格

请实现一个函数，把字符串 `s` 中的每个空格替换成"%20"。

```
输入：s = "We are happy."
输出："We%20are%20happy."
```

1. go strings库中Replace和ReplaceAll函数

```go
func replaceSpace(s string) string {
	return strings.Replace(s," ","%20",-1) 
}
func replaceSpace(s string) string {
	return strings.ReplaceAll(s, " ", "%20")
}
```

2. go strings库中Builder
```go
func replaceSpace(s string) string {
	var newString strings.Builder
	for _, i := range s {
		if i == ' ' {
			newString.WriteString("%20")
		} else {
			newString.WriteString(string(i))
		}
	}
	return newString.String()
}

func replaceSpace(s string) string {
	var newString strings.Builder
	for i := range s {
		if s[i] == ' ' {
			newString.WriteString("%20")
		} else {
			newString.WriteByte(s[i])
		}
	}
	return newString.String()
}
```


3. 

