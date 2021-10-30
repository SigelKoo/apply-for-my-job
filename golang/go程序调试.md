# delve（dlv）

dlv是Golang实现的Golang调试器，目前dlv对windows平台的支持似乎不是很好，我在windows平台调试，dlv无法找到目标程序的源代码，因此建议在Linux平台下调试Golang程序时使用。使用dlv前，需在本地通过`go get github.com/derekparker/delve/cmd/dlv`进行安装。

1. 带参数启动程序（`dlv exec ./GoDbg -- arg1 arg2`）

   ```
   [lday@alex GoDbg]$ dlv exec ./GoDbg -- arg1 arg2 
   Type 'help' for list of commands.
   (dlv) 
   ```

2. 在main函数上设置断点（`b`）

   ```
   (dlv) b main.main
   Breakpoint 1 set at 0x40101b for main.main() ./main.go:9
   ```

3. 启动调试，断点后继续执行（`c`）

   ```
   (dlv) c
   > main.main() ./main.go:9 (hits goroutine(1):1 total:1) (PC: 0x40101b)
        4:        "GoWorks/GoDbg/mylib"
        5:        "fmt"
        6:        "os"
        7:    )
        8:    
   =>   9:    func main() {
       10:        fmt.Println("Golang dbg test...")
       11:    
       12:        var argc = len(os.Args)
       13:        var argv = append([]string{}, os.Args...)
       14:    
   ```

4. 在文件dbgTest.go上通过行号设置断点（`b`）

   ```
   (dlv) b dbgTest.go:17
   Breakpoint 2 set at 0x457f51 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:17
   (dlv) b dbgTest.go:23
   Breakpoint 3 set at 0x4580d0 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:23
   (dlv) b dbgTest.go:26
   Breakpoint 4 set at 0x458123 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:26
   (dlv) b dbgTest.go:29
   Breakpoint 5 set at 0x458166 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:29
   ```

5. 显示所有断点列表（`bp`）

   ```
   (dlv) bp
   Breakpoint unrecovered-panic at 0x429690 for runtime.startpanic() /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/panic.go:524 (0)
   Breakpoint 1 at 0x40101b for main.main() ./main.go:9 (1)
   Breakpoint 2 at 0x457f51 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:17 (0)
   Breakpoint 3 at 0x4580d0 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:23 (0)
   Breakpoint 4 at 0x458123 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:26 (0)
   Breakpoint 5 at 0x458166 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:29 (0)
   ```

   dlv似乎没有提供类似gdb`dis x`，禁止某个断点的功能，在文档中暂时没有查到。不过这个功能用处不大。

6. 删除某个断点（`clear x`）

   ```
   (dlv) clear 5
   Breakpoint 5 cleared at 0x458166 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:29
   (dlv) bp
   Breakpoint unrecovered-panic at 0x429690 for runtime.startpanic() /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/panic.go:524 (0)
   Breakpoint 1 at 0x40101b for main.main() ./main.go:9 (1)
   Breakpoint 2 at 0x457f51 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:17 (0)
   Breakpoint 3 at 0x4580d0 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:23 (0)
   Breakpoint 4 at 0x458123 for GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:26 (0)
   ```

7. 显示当前运行的代码位置（`ls`）

   ```
   (dlv) ls
   > GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:17 (hits goroutine(1):1 total:1) (PC: 0x457f51)
       12:        C map[int]string
       13:        D []string
       14:    }
       15:    
       16:    func DBGTestRun(var1 int, var2 string, var3 []int, var4 MyStruct) {
   =>  17:        fmt.Println("DBGTestRun Begin!\n")
       18:        waiter := &sync.WaitGroup{}
       19:    
       20:        waiter.Add(1)
       21:        go RunFunc1(var1, waiter)
       22:    
   ```

8. 查看当前调用栈信息（`bt`）

   ```
   (dlv) bt
   0  0x0000000000457f51 in GoWorks/GoDbg/mylib.DBGTestRun
      at ./mylib/dbgTest.go:17
   1  0x0000000000401818 in main.main
      at ./main.go:27
   2  0x000000000042aefb in runtime.main
      at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/proc.go:188
   3  0x0000000000456df0 in runtime.goexit
      at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/asm_amd64.s:1998
   ```

9. 输出变量信息（`print/p`）

   ```
   (dlv) print var1
   1
   (dlv) print var2
   "golang dbg test"
   (dlv) print var3
   []int len: 3, cap: 3, [1,2,3]
   (dlv) print var4
   GoWorks/GoDbg/mylib.MyStruct {
       A: 1,
       B: "golang dbg my struct field B",
       C: map[int]string [
           1: "value1", 
           2: "value2", 
           3: "value3", 
       ],
       D: []string len: 3, cap: 3, ["D1","D2","D3"],}
   ```

   **类比gdb调试，我们看到，之前我们使用gdb进行调试时，发现gdb在此时无法输出var3, var4的内容，而dlv可以**

10. 在第n层调用栈上执行相应指令（`frame n cmd`）

    ```
    (dlv) frame 1 ls
        22:        var4.A = 1
        23:        var4.B = "golang dbg my struct field B"
        24:        var4.C = map[int]string{1: "value1", 2: "value2", 3: "value3"}
        25:        var4.D = []string{"D1", "D2", "D3"}
        26:    
    =>  27:        mylib.DBGTestRun(var1, var2, var3, var4)
        28:        fmt.Println("Golang dbg test over")
        29:    }
    ```

    `frame 1 ls`将显示程序在第1层调用栈上的具体实行位置

11. 查看goroutine的信息（`goroutines`）
    当我们执行到`dbgTest.go:26`时，我们已经启动了两个goroutine

    ```
    (dlv) 
    > GoWorks/GoDbg/mylib.DBGTestRun() ./mylib/dbgTest.go:26 (hits goroutine(1):1 total:1) (PC: 0x458123)
        21:        go RunFunc1(var1, waiter)
        22:    
        23:        waiter.Add(1)
        24:        go RunFunc2(var2, waiter)
        25:    
    =>  26:        waiter.Add(1)
        27:        go RunFunc3(&var3, waiter)
        28:    
        29:        waiter.Add(1)
        30:        go RunFunc4(&var4, waiter)
        31:    
    ```

    此时我们来查看程序的goroutine状态信息

    ```
    (dlv) goroutines
    [6 goroutines]
    * Goroutine 1 - User: ./mylib/dbgTest.go:26 GoWorks/GoDbg/mylib.DBGTestRun (0x458123) (thread 9022)
      Goroutine 2 - User: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/proc.go:263 runtime.gopark (0x42b2d3)
      Goroutine 3 - User: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/proc.go:263 runtime.gopark (0x42b2d3)
      Goroutine 4 - User: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/proc.go:263 runtime.gopark (0x42b2d3)
      Goroutine 5 - User: ./mylib/dbgTest.go:39 GoWorks/GoDbg/mylib.RunFunc1 (0x4583eb) (thread 9035)
      Goroutine 6 - User: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/format.go:130 fmt.(*fmt).padString (0x459545)
    ```

    从输出的信息来看，先启动的goroutine 5，执行`RunFunc1`，此时还没有执行`fmt.Printf`，而后启动的goroutine 6，执行`RunFunc2`，则已经进入到`fmt.Printf`的内部调用过程中了

12. 进一步查看goroutine信息（`goroutine x`）
    接第11步的操作，此时我想查看goroutine 6的具体执行情况，则执行`goroutine 6`

    ```
    (dlv) goroutine 6
    Switched from 1 to 6 (thread 9022)
    ```

    在此基础上，执行`bt`，则可以看到当前goroutine的调用栈情况

    ```
    (dlv) bt
     0  0x0000000000454730 in runtime.systemstack_switch
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/asm_amd64.s:245
     1  0x000000000040f700 in runtime.mallocgc
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/malloc.go:643
     2  0x000000000040fc43 in runtime.rawmem
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/malloc.go:809
     3  0x000000000043c2a5 in runtime.growslice
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/slice.go:95
     4  0x000000000043c015 in runtime.growslice_n
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/slice.go:44
     5  0x0000000000459545 in fmt.(*fmt).padString
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/format.go:130
     6  0x000000000045a13f in fmt.(*fmt).fmt_s
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/format.go:322
     7  0x000000000045e905 in fmt.(*pp).fmtString
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:518
     8  0x000000000046200f in fmt.(*pp).printArg
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:797
     9  0x0000000000468a8d in fmt.(*pp).doPrintf
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:1238
    10  0x000000000045c654 in fmt.Fprintf
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:188
    ```

    此时输出了10层调用栈，但似乎最原始的我自身程序dbgTest.go的调用栈没有输出， 可以通过`bt`加depth参数，设定bt的输出深度，进而找到我们自己的调用栈，例如`bt 13`

    ```
    (dlv) bt 13
    ...
    10  0x000000000045c654 in fmt.Fprintf
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:188
    11  0x000000000045c74b in fmt.Printf
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/print.go:197
    12  0x000000000045846f in GoWorks/GoDbg/mylib.RunFunc2
        at ./mylib/dbgTest.go:50
    13  0x0000000000456df0 in runtime.goexit
        at /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/asm_amd64.s:1998
    ```

    我们看到，我们自己dbgTest.go的调用栈在第12层。当前goroutine已经不再我们自己的调用栈上，而是进入到系统函数的调用中，在这种情况下，使用gdb进行调试时，我们发现，此时我们没有很好的方法能够输出我们需要的调用栈变量信息。**dlv可以!**此时只需简单的通过`frame x cmd`就可以输出我们想要的调用栈信息了

    ```
    (dlv) frame 12 ls
        45:        time.Sleep(10 * time.Second)
        46:        waiter.Done()
        47:    }
        48:    
        49:    func RunFunc2(variable string, waiter *sync.WaitGroup) {
    =>  50:        fmt.Printf("var2:%v\n", variable)
        51:        time.Sleep(10 * time.Second)
        52:        waiter.Done()
        53:    }
        54:    
        55:    func RunFunc3(pVariable *[]int, waiter *sync.WaitGroup) {
    (dlv) frame 12 print variable 
    "golang dbg test"
    (dlv) frame 12 print waiter
    *sync.WaitGroup {
        state1: [12]uint8 [0,0,0,0,2,0,0,0,0,0,0,0],
        sema: 0,}
    ```

    多好的功能啊！

13. 查看当前是在哪个goroutine上（`goroutine`）
    当使用`goroutine`不带参数时，dlv就会显示当前goroutine信息，这可以帮助我们在调试时确认是否需要做goroutine切换

    ```
    (dlv) goroutine
    Thread 9022 at ./mylib/dbgTest.go:26
    Goroutine 6:
        Runtime: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/runtime/asm_amd64.s:245 runtime.systemstack_switch (0x454730)
        User: /home/lday/Tools/Dev_Tools/Go_Tools/go_1_6_2/src/fmt/format.go:130 fmt.(*fmt).padString (0x459545)
        Go: ./mylib/dbgTest.go:26 GoWorks/GoDbg/mylib.DBGTestRun (0x458123)
    ```

## dlv前端(gdlv)

dlv提供了类似gdb的cli调试系统，而有第三方还提供了dlv的GUI前端([gdlv](https://github.com/aarzilli/gdlv))，对于那些习惯了使用GUI进行调试的人来说，结合gdlv和dlv，调试会更加方便。gdlv有个问题是：他无法在xwindows server上运行，只能在server本地运行。
[![img](https://raw.githubusercontent.com/aarzilli/gdlv/master/doc/screen.png)](https://raw.githubusercontent.com/aarzilli/gdlv/master/doc/screen.png)