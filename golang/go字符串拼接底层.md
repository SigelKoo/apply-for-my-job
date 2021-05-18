# go字符串拼接底层

- +拼接

  字符串在Go语言中是不可变类型，占用内存大小是固定的，当使用 +拼接 2 个字符串时，生成一个新的字符串，那么就需要开辟一段新的空间，新空间的大小是原来两个字符串的大小之和。拼接第三个字符串时，再开辟一段新空间，新空间大小是三个字符串大小之和，以此类推。

  ```go
  func plusConcat(n int, str string) string {
  	s := ""
  	for i := 0; i < n; i++ {
  		s += str
  	}
  	return s
  }
  ```

- strings.Builder

  切片[]byte的内存是以倍数申请的。例如，初始大小为0，当第一次写入大小为10 byte的字符串时，则会申请大小为16 byte的内存（大于10 byte的2的指数），第二次写入10 byte时，内存不够，则申请32 byte的内存，第三次写入内存足够，则不申请新的，以此类推。

  ```go
  func builderConcat(n int, str string) string {
  	var builder strings.Builder
  	for i := 0; i < n; i++ {
  		builder.WriteString(str)
  	}
  	return builder.String()
  }
  ```

- bytes.Buffer

  strings.Builder性能比bytes.Buffer略快10%，区别为bytes.Buffer转化为字符串时重新申请了一块空间，存放生成的字符串变量，而strings.Builder直接将底层的[]byte转换成字符串类型返回了回来。

  ```go
  func bufferConcat(n int, s string) string {
  	buf := new(bytes.Buffer)
  	for i := 0; i < n; i++ {
  		buf.WriteString(s)
  	}
  	return buf.String()
  }
  ```

  