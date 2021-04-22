六线程顺序输出ABC
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	A := make(chan int, 1)
	B := make(chan int, 1)
	C := make(chan int, 1)
	D := make(chan int, 1)
	E := make(chan int, 1)
	F := make(chan int, 1)
	wg.Add(6)

	count := 100

	A <- 0
	
	go func(chan int, int, chan int) {
		ACount := 0
		for ACount < count {
			select {
			case <- A:
				fmt.Println("A")
				ACount++
				B <- 0
			}
		}
		defer wg.Done()
	}(A, count, B)

	go func(chan int, int, chan int) {
		BCount := 0
		for BCount < count {
			select {
			case <- B:
				fmt.Println("B")
				BCount++
				C <- 0
			}
		}
		defer wg.Done()
	}(B, count, C)

	go func(chan int, int, chan int) {
		CCount := 0
		for CCount < count {
			select {
			case <- C:
				fmt.Println("C")
				CCount++
				D <- 0
			}
		}
		defer wg.Done()
	}(C, count, D)

	go func(chan int, int, chan int) {
		DCount := 0
		for DCount < count {
			select {
			case <- D:
				fmt.Println("D")
				DCount++
				E <- 0
			}
		}
		defer wg.Done()
	}(D, count, E)

	go func(chan int, int, chan int) {
		ECount := 0
		for ECount < count {
			select {
			case <- E:
				fmt.Println("E")
				ECount++
				F <- 0
			}
		}
		defer wg.Done()
	}(E, count, F)

	go func(chan int, int, chan int) {
		FCount := 0
		for FCount < count {
			select {
			case <- F:
				fmt.Println("F")
				FCount++
				A <- 0
			}
		}
		defer wg.Done()
	}(F, count, A)

	wg.Wait()
}
```

三线程输出0~100

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	A := make(chan int, 1)
	B := make(chan int, 1)
	C := make(chan int, 1)
	wg.Add(3)

	count := 0

	A <- 0

	go func(chan int, int, chan int) {
		for count < 100 {
			select {
			case <- A:
				if count < 100 {
					fmt.Println("A", count)
				}
				count++
				B <- 0
			}
		}
		defer wg.Done()
	}(A, count, B)

	go func(chan int, int, chan int) {
		for count < 100 {
			select {
			case <- B:
				if count < 100 {
					fmt.Println("B", count)
				}
				count++
				C <- 0
			}
		}
		defer wg.Done()
	}(B, count, C)

	go func(chan int, int, chan int) {
		for count < 100 {
			select {
			case <- C:
				if count < 100 {
					fmt.Println("C", count)
				}
				count++
				A <- 0
			}
		}
		defer wg.Done()
	}(C, count, A)

	wg.Wait()
}
```

