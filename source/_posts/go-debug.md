---
title: 'Go的调试工具:gdb vs dlv'
tags:
  - Go
categories:
  - Go
date: '2019-4-18 19:45'
abbrlink: 8063
---

GoLand编辑器虽然很强大，但是在展示内存及堆栈信息这一块还是比较的弱，有可能是我的姿势不对，所以，开始切入了gdb调试，但是gdb踩到了坑，并没有解决，也就引发了gdb与dlv的对比了

<!--more-->

## gdb

### 安装

~~~shell
yum install ncures-devel
wget http://ftp.gnu.org/gnu/gdb/gdb-8.2.tar.gz
tar zxf gdb-8.2.tar.gz
cd gdb-8.2
make && make install
~~~

### 进入调试

~~~
go build -gcflags '-N -l' main.go
gdb main
~~~

### 使用

1. 启动调试程序（`gdb`）

   ```
   [lday@alex GoDbg]$ gdb ./GoDbg
   ```

2. 在main函数上设置断点（`b`）

   ```
   (gdb) b main.main
   Breakpoint 1 at 0x401000: file /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go, line 9.
   ```

3. 带参数启动程序（`r`）

   ```
   (gdb) r arg1 arg2
   Starting program: /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/GoDbg arg1 arg2
   [New LWP 8412]
   [New LWP 8413]
   [New LWP 8414]
   [New LWP 8415]
   
   Breakpoint 1, main.main () at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
   9    func main() {
   ```

4. 在文件dbgTest.go上通过行号设置断点（`b`）

   ```
   (gdb) b dbgTest.go:16
   Breakpoint 3 at 0x457960: file /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go, line 16.
   ```

5. 查看断点设置情况（`info b`）

   ```
   (gdb) info b
   Num     Type           Disp Enb Address            What
   1       breakpoint     keep y   0x0000000000401000 in main.main 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
       breakpoint already hit 1 time
   2       breakpoint     keep y   0x0000000000401000 in main.main 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
       breakpoint already hit 1 time
   3       breakpoint     keep y   0x0000000000457960 in GoWorks/GoDbg/mylib.DBGTestRun 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go:16
   ```

6. 禁用断点（`dis n`）

   ```
   (gdb) dis 1   
   (gdb) info b
   Num     Type           Disp Enb Address            What
   1       breakpoint     keep n   0x0000000000401000 in main.main 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
       breakpoint already hit 1 time
   2       breakpoint     keep y   0x0000000000401000 in main.main 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
       breakpoint already hit 1 time
   3       breakpoint     keep y   0x0000000000457960 in GoWorks/GoDbg/mylib.DBGTestRun 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go:16
   ```

7. 删除断点（`del n`）

   ```
   (gdb) del 1
   (gdb) info b
   Num     Type           Disp Enb Address            What
   2       breakpoint     keep y   0x0000000000401000 in main.main 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:9
       breakpoint already hit 1 time
   3       breakpoint     keep y   0x0000000000457960 in GoWorks/GoDbg/mylib.DBGTestRun 
                                                      at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go:16
   ```

8. 断点后继续执行（`c`）

   ```
   (gdb) c
   Continuing.
   Golang dbg test...
   argc:3
   argv:[/home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/GoDbg arg1 arg2]
   
   Breakpoint 3, GoWorks/GoDbg/mylib.DBGTestRun (var1=1, var2="golang dbg test")
       at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go:16
   16    func DBGTestRun(var1 int, var2 string, var3 []int, var4 MyStruct) {
   (gdb) 
   ```

9. 显示代码（`l`）

   ```
   (gdb) l
   11        B string
   12        C map[int]string
   13        D []string
   14    }
   15    
   16    func DBGTestRun(var1 int, var2 string, var3 []int, var4 MyStruct) {
   17        fmt.Println("DBGTestRun Begin!\n")
   18        waiter := &sync.WaitGroup{}
   19    
   20        waiter.Add(1)
   ```

10. 单步执行（`n`）

    ```
    (gdb) n
    DBGTestRun Begin!
    
    18        waiter := &sync.WaitGroup{}
    ```

11. 打印变量信息（`print/p`）
    在进入DBGTestRun的地方设置断点(`b dbgTest.go:16`)，进入该函数后，通过p命令显示对应变量：

    ```
    (gdb) l 17
    12        C map[int]string
    13        D []string
    14    }
    15    
    16    func DBGTestRun(var1 int, var2 string, var3 []int, var4 MyStruct) {
    17        fmt.Println("DBGTestRun Begin!\n")
    18        waiter := &sync.WaitGroup{}
    19    
    20        waiter.Add(1)
    21        go RunFunc1(var1, waiter)
    (gdb) p var1 
    $3 = 1
    (gdb) p var2
    $4 = "golang dbg test"
    (gdb) p var3
    No symbol "var3" in current context.
    ```

    **从上面的输出我们可以看到一个很奇怪的事情，虽然DBGTestRun有4个参数传入，但是，似乎var3和var4 gdb无法识别，在后续对dlv的实验操作中，我们发现，dlv能够识别var3， var4.**

12. 查看调用栈（`bt`），切换调用栈（`f n`），显示当前栈变量信息

    ```
    (gdb) bt
    #0  GoWorks/GoDbg/mylib.DBGTestRun (var1=1, var2="golang dbg test")
        at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/mylib/dbgTest.go:17
    #1  0x00000000004018c2 in main.main () at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:27
    (gdb) f 1
    #1  0x00000000004018c2 in main.main () at /home/lday/Works/Go_Works/GoLocalWorks/src/GoWorks/GoDbg/main.go:27
    27        mylib.DBGTestRun(var1, var2, var3, var4)
    (gdb) l
    22        var4.A = 1
    23        var4.B = "golang dbg my struct field B"
    24        var4.C = map[int]string{1: "value1", 2: "value2", 3: "value3"}
    25        var4.D = []string{"D1", "D2", "D3"}
    26    
    27        mylib.DBGTestRun(var1, var2, var3, var4)
    28        fmt.Println("Golang dbg test over")
    29    }
    (gdb) print var1 
    $5 = 1
    (gdb) print var2
    $6 = "golang dbg test"
    (gdb) print var3
    $7 =  []int = {1, 2, 3}
    
    (gdb) print var4
    $8 = {A = 1, B = "golang dbg my struct field B", C = map[int]string = {[1] = "value1", [2] = "value2", [3] = "value3"}, 
    D =  []string = {"D1", "D2", "D3"}}
    ```



## dlv(推荐)

### 安装

~~~
go get -u github.com/go-delve/delve/cmd/dlv
~~~

###  进入调试

~~~
dlv debug main.go
~~~

### 使用

~~~
    args ------------------------ Print function arguments.
    break (alias: b) ------------ Sets a breakpoint.
    breakpoints (alias: bp) ----- Print out info for active breakpoints.
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    clear ----------------------- Deletes breakpoint.
    clearall -------------------- Deletes multiple breakpoints.
    condition (alias: cond) ----- Set breakpoint condition.
    config ---------------------- Changes configuration parameters.
    continue (alias: c) --------- Run until breakpoint or program termination.
    deferred -------------------- Executes command in the context of a deferred call.
    disassemble (alias: disass) - Disassembler.
    down ------------------------ Move the current frame down.
    edit (alias: ed) ------------ Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ------ Exit the debugger.
    frame ----------------------- Set the current frame, or execute command on a different frame.
    funcs ----------------------- Print list of functions.
    goroutine ------------------- Shows or changes current goroutine
    goroutines ------------------ List program goroutines.
    help (alias: h) ------------- Prints the help message.
    libraries ------------------- List loaded dynamic libraries
    list (alias: ls | l) -------- Show source code.
    locals ---------------------- Print local variables.
    next (alias: n) ------------- Step over to next source line.
    on -------------------------- Executes a command when a breakpoint is hit.
    print (alias: p) ------------ Evaluate an expression.
    regs ------------------------ Print contents of CPU registers.
    restart (alias: r) ---------- Restart process.
    set ------------------------- Changes the value of a variable.
    source ---------------------- Executes a file containing a list of delve commands
    sources --------------------- Print list of source files.
    stack (alias: bt) ----------- Print stack trace.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout --------------------- Step out of the current function.
    thread (alias: tr) ---------- Switch to the specified thread.
    threads --------------------- Print out info for every traced thread.
    trace (alias: t) ------------ Set tracepoint.
    types ----------------------- Print list of types
    up -------------------------- Move the current frame up.
    vars ------------------------ Print package variables.
    whatis ---------------------- Prints type of an expression.
~~~

## 总结

1. dlv对goroutine的支持更好，我使用gdb的没有找到goroutine的调试方法，可能姿势不对

2. gdb对于局部引用变量无法调试，dlv不会

   ~~~go
   func main() {
     i := 10
     j := &i
   }
   ~~~

   如上代码，gdb 使用 `p j` 打印变量j的时候报错，dlv却可以
   
3. dlv无法调试interface等Go内部实现的一些结构，gdb是可以的



## 参考

[《Golang程序调试工具介绍(gdb vs dlv)》](http://lday.me/2017/02/27/0005_gdb-vs-dlv/)