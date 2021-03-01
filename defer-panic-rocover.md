# defer panic rocover

[原文出处](https://blog.golang.org/defer-panic-and-recover)
翻译：姚雨辰
时间：2021-03-01


`defer`语句可以把函数调用压入一个列表里，函数返回后执行该列表。
同步常用于简化各种函数返回后的清理操作。

例如，将一个文件的内容复制到另一个函数：

    ```go
    func CopyFile(dstName, srcName string) (written int64, err error) {
        src, err := os.Open(srcName)
        if err != nil {
            return
        }

        dst, err := os.Create(dstName)
        if err != nil {
            return
        }

        written, err = io.Copy(dst, src)
        dst.Close()
        src.Close()
        return
    }
    ```

上例中，如果`os.Create`调用失败，该函数将直接返回，不关闭源文件。
通过`defer`语句可以更优雅的解决这个问题：

    ```go
    func CopyFile(dstName, srcName string) (written int64, err error) {
        src, err := os.Open(srcName)
        if err != nil {
            return
        }
        defer src.Close()

        dst, err := os.Create(dstName)
        if err != nil {
            return
        }
        defer dst.Close()

        return io.Copy(dst, src)
    }
    ```

我们可以在文件打开之后立即执行`defer`语句进行关闭，这将在函数返回后自动关闭。

---

`defer`语句的行为时直切且可预测的。
下面是三个简单准则：

1.  A deferred function's arguments are evaluated when the defer statement is evaluated.

    此例中`defer`会打印`0`而不是`1`：

    ```go
    func a() {
        i := 0
        defer fmt.Println(i)
        i++
        return
    }
    ```

2.  Deferred function calls are executed in Last In First Out order after the surrounding function returns.

    此例打印`3210`：

    ```go
    func b() {
        for i := 0; i < 4; i++ {
            defer fmt.Print(i)
        }
    }
    ```

3.  Deferred functions may read and assign to the returning function's named return values.

    此例返回`2`：

    ```go
    func c() (i int) {
        defer func() { i++ }()
        return 1
    }
    ```

者可以非常方便的确定函数返回的错误的类型；看下面的例子。

`Panic`是一个内置函数，可停止常规控制流并开始`panicking`。
当函数`F`调用`panic`时，`F`的执行停止，`F`中`defer`函数都将正常执行，然后`F`返回其调用方。
对调用者而言，`F`表现为调用`panic`。
该过程将继续执行堆栈，直到当前`goroutine`中的所有函数返回为止，此时程序崩溃。
紧急事件可以通过直接调用`panic`来启动。
它们也可能是由`runtime error`引起的，例如越界数组访问。


`Recover`是一个内置函数，可以重新获得对`panic goroutine`的控制。
`Recover`仅在`defer`内部有用。
在正常执行期间，恢复调用将返回`nil`，并且没有其他动作。
如果当前`goroutine`处于`panic`，则调用`recover`会捕获提供给`panic`的值并恢复正常执行。


下面的例子示范了`defer`和`panic`的机理：

    ```go
    package main

    import "fmt"

    func main() {
        f()
        fmt.Println("Returned normally from f.")
    }

    func f() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Println("Recovered in f", r)
            }
        }()
        fmt.Println("Calling g.")
        g(0)
        fmt.Println("Returned normally from g.")
    }

    func g(i int) {
        if i > 3 {
            fmt.Println("Panicking!")
            panic(fmt.Sprintf("%v", i))
        }
        defer fmt.Println("Defer in g", i)
        fmt.Println("Printing in g", i)
        g(i + 1)
    }
    ```

函数`g`获取参数`i`，如果`i > 3`就`panic`，否则就用`i+1`调用自身。
函数`f`，`defer` 了一个调用`revover`并打印的函数。

程序的输出：

    ```go
    Calling g.
    Printing in g 0
    Printing in g 1
    Printing in g 2
    Printing in g 3
    Panicking!
    Defer in g 3
    Defer in g 2
    Defer in g 1
    Defer in g 0
    Recovered in f 4
    Returned normally from f.
    ```

如果移除函数`f`里的`defered function`，`panic`没有被`recovered`后到达`goroutine`的`call stack`的顶，结束程序。
这是修改后的程序：

    ```go
    Calling g.
    Printing in g 0
    Printing in g 1
    Printing in g 2
    Printing in g 3
    Panicking!
    Defer in g 3
    Defer in g 2
    Defer in g 1
    Defer in g 0
    panic: 4

    panic PC=0x2a9cd8
    [stack trace omitted]
    ```

`Go`的惯例时集是某一个包用了`panic`，外部的API还是会显示的展示出错误返回值。

还可以用`defer`来释放锁：

    ```go
    mu.Lock()
    defer mu.Unlock()
    ```

打印footer：

    ```go
    printHeader()
    defer printFooter()
    ```

总之，`defer`语句为控制流提供了一个不寻常且十分有效的机制。
它可用来模拟一系列由其他编程语言实现的专用结构的特征。