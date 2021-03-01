# share memory by communicating

[原文出处](https://blog.golang.org/codelab-share)
翻译：姚雨辰
时间：2021-03-01


传统线程模型要求程序员用共享内存的方式来通信。

比较典型的方法是：共享被锁保护的数据结构，线程通过竞争来获得这些锁住的数据。
在某些情况下，这让线程安全的数据结构用起来更简单。例如`Python`的`Queue`。

`Go`的并发基本元--`goroutines`和`chennels`--为构建并发软件提供了一种优雅且明确的方式。
除了显示地调用来调解对数据的访问，`Go`更加提倡使用`channels`来在`goroutines`之间传递数据的引用。
这种方法确保了同一时刻只有一个`goroutine`能获得数据的访问权。
这个章节在文档[Effective Go]()中做了总结：

​	*Do not communicate by sahring memory;instead,share memory by communicating.*


考虑一个`poll`一个URL列表的程序。
在传统的线程环境中，大家应该会这样组织自己的数据：

    ```go
    type Resource struct {
        url        string
        polling    bool
        lastPolled int64
    }

    type Resources struct {
        data []*Resource
        lock *sync.Mutex
    }    
    ```

一个`Poller`函数可能是这样（运行在不同线程）：

    ```go
    func Poller(res *Resources) {
        for {
            // get the least recently-polled Resource
            // and mark it as being polled
            res.lock.Lock()
            var r *Resource
            for _, v := range res.data {
                if v.polling {
                    continue
                }
                if r == nil || v.lastPolled < r.lastPolled {
                    r = v
                }
            }
            if r != nil {
                r.polling = true
            }
            res.lock.Unlock()
            if r == nil {
                continue
            }

            // poll the URL

            // update the Resource's polling and lastPolled
            res.lock.Lock()
            r.polling = false
            r.lastPolled = time.Nanoseconds()
            res.lock.Unlock()
        }
    }
    ```

这个函数看起来有一页，并且需要完善更多的细节。
甚至没有包含`polling`的逻辑，也无法优雅的处理棘手的资源池。


让我们来看看如何用`Go`的方式实现同样功能。
在本例中，`poller`从输入`channel`接收需要`poll`的资源，并在处理完之后将其发送到输出`channel`。

    ```go
    type Resource string

    func Poller(in, out chan *Resource) {
        for r := range in {
            // poll the URL

            // send the processed Resource to out
            out <- r
        }
    }
    ```

没有了复杂的逻辑与繁杂的数据。当然，剩下来的都是很重要的部分。
[完整代码](https://golang.org/doc/codewalk/sharemem/)

这个例子也许会让你对`Go`的魅力有一个大概的领会了。