#  go maps in action

[原文出处](https://blog.golang.org/maps)
翻译：姚雨辰
时间：2021-03-01

## 介绍

哈希表是计算机科学中最有用的数据结构之一。即使许多哈希表的实现都伴随着性能上的差异，但总的来说都提供了快速查找、添加、删除的功能。`Go`也提供了实现哈希表的内置类型--`map`。

## 声明和初始化

`Go`语言里的`map`长这样：
    
    ```go
    map[KeyType]ValueType
    ```
`KeyType`可以是任何**可比较类型**， 而`ValueType`可以是任何类型（包含`map`）。





下边声明的`m`就是一个有`string`键和`int`值的`map`：

    ```go
    var m map[string]int
    ```

`map`类型也是引用类型（像pointers和slices），所以上述`map`的值其实是`nil`，而不是指向一个初始化后的`map`。

一个`nil map`在读的时候表现的和空`map`一样，但如果你想向`nil map`进行写操作，则会引发`runtime panic`！


像下面这样使用内置函数`make`来初始化一个`map`:

    ```go
    m = make(map[string]int)
    ```
`make`函数分配并初始化一个哈希表数据结构，返回一个指向该哈希表的`map`。

*该数据结构涉及到`runtime`的一些实现细节且不由语言本身决定。我们目前只涉及使用，不关注实现。*



## 使用`map`


`Go`为`map`提供了我们常见的语法。

如下语句将键`route`的值设为了`66`：

    ```go
    m["route"] = 66
    ```

下面语句将键`route`的值取出并赋给新变量`i`：

    ```go
    i := m["route"]
    ```

如果你所请求的键并不存在，则会取出一个“零值”。具体是什么要取决于该`map`的值的类型：

    ```go
    j := m["root"]
    // j == 0
    // map[string]int
    ```

内建函数`len`返回`map`中的键值对数目：

    ```go
    n := len(m)
    ```

内建函数`delete`移除你指定的键值对：

    ```go
    delete(m, "route")
    ```

`delete`函数没有返回值，如果指定键不存在，则`delete`什么也不做。


如下“二值赋值测试”可用于检验键的存在性：

    ```go
    i, ok := m["route"]
    ```

上述赋值语句中，`i`被赋为键`route`对应的值（不存在则为“零值”），`ok`则被赋一个`bool`值，该键存在时为`true`。

如果只想测试键是否存在：

    ```go
    _, ok := m["route"]
    ```

遍历`map`，可以用关键词`range`：

    ```go
    for key, value := range m {
        fmt.Println("key:", key, "Value:"， value)
    }
    ```

用一组数据初始化一个`map`：

    ```go
    commits := map[string]int{
        "src" : 3711,
        "r"   : 2138,
        "gri" : 1908,
        "adg" : 912,
    }
    ```

可以用同样的语法来初始化一个空`map`，在功能上与用`make`完全一致：

    ```go
    m = map[string]int{}
    ```



## 利用零值


当键不存在的时候取回一个“零值”很实用。

例如，一个`bool`值的`map`可以被当成类`set`数据结构来使用（`bool`类型的“零值”是`false`）。

下例遍历了一个节点为`Node`的链表并打印了值，其中使用了存储指向`Node`的指针的`map`来监测链表中的“环”：

    ```go
    type Node struct {
        Next  *Node
        Value interface{}
    }
    var first *Node

    visited := make(map[*Node]bool)
    for n := first; n != nil; n = n.Next {
        if visited[n] {
            fmt.Println("cycle detected")
            break
        }
        visited[n] = true
        fmt.Println(n.Value)
    }
    ```

如果`n`被检访问过了，则`visited[n]`为`true`。
这时我们不用再使用“二值测试”的形式来测试`n`是否在`map`里了，默认的“零值”已经提我们做过这些了。

感觉这段翻译的不太对劲，贴一下原文的表述：

The expression visited[n] is true if n has been visited, or false if n is not present.There's no need to use the two-value form to test for the presence of n in the map; the zero value default does it for us.

---

另一个有用的关于“零值”实例是`a map of slices`。
往`nil slice`后附加元素会分配一个新的`slice`，所以“往`a map of slices`后追加值”是一个单行操作；根本就没有检查键是否存在的必要。

在下面的例子里，`slice people`是由`Person`值构成的。每个`Person`都有一个`Name`和一个`slice Likes`。
该例创建了一个`map`来组织每一个`like`与喜欢这样东西的人的`slice`。

    ```go
        type Person struct {
        Name  string
        Likes []string
    }
    var people []*Person

    likes := make(map[string][]*Person)
    for _, p := range people {
        for _, l := range p.Likes {
            likes[l] = append(likes[l], p)
        }
    }
    ```

打印喜欢`cheese`的人的清单：

    ```go
        for _, p := range likes["cheese"] {
        fmt.Println(p.Name, "likes cheese.")
    }
    ```

打印喜欢`bacon`的人数：

    ```go
        fmt.Println(len(likes["bacon"]), "people like bacon.")
    ```

**注意：**
    由于`range`和`len`都把`nil slice`当作长度为`0`的`slice`来对待，所以即使没有人喜欢`cheese`或是`bacon`，最后两个例子也会正常工作。




## 键的类型


正如之前提到的，`map`的键可以是任意可比较类型。
[语言细则](https://golang.org/ref/spec#Comparison_operators)已经详细定义了，不过我们直接说吧：`boolean`,`numeric`,`string`,`pointer`,`channel`,和只含有前述类型的`interface`,`struct`,`array`。
显然，`slice`,`map`,`function`不在其中;这类类型不能被`==`比较，所以大概也不可以用作键。

显然，`string`,`int`和其他基础数据类型都可以作为`map`的键，但让人不太能接受的可能是`struct`键。
`struct`可以从许多维度上被用为键。

例如，下边的这个`map of maps`可以被用来统计网页点击数：
    
    ```go
    hits := make(map[string]map[string]int)
    ```

这是一个存储（`string`，`存储（string，int）的map  `）的`map`。

外层`map`的每个键都是网页路径，这个网页存储内层的`map`。每个内层的`map`的键是包含两个字母的国家代号。
底下的表达式取回澳大利亚人加载文档页的次数：

    ```go
    n := hits["/doc/"]["au"]
    ```

不幸的是，这种方法在添加数据是变得十分不方便，对于每个给出的外层键，你都需要检测内层键的存在性，并在必要的时候创建：

    ```go
    func add(m map[string]map[string]int, path, country string) {
        mm, ok := m[path]
        if !ok {
            mm = make(map[string]int)
            m[path] = mm
        }
        mm[country]++
    }
    add(hits, "/doc/", "au")
    ```

但是，如果用简单的`map`加上一个`struct`，则免去了上述的所有麻烦：

    ```go
    type Key struct {
        Path, Country string
    }
    hits := make(map[key]int)
    ```

当一个越南人访问网页，添加访问次数变成了单行操作：

    ```go
    hits[Key{"/", "vn"}]++
    ```

查看信息也变得十分方便了：

    ```go
    n := hits[Key{"/ref/spec", "ch"}]
    ```




## 并发


[并发使用`Map`是不安全的：](https://golang.org/doc/faq#atomic_maps)当你同时读写`map`时，可能发生的情况是未定义的。

如果你需要在并发执行的`goroutines`间进行读、写，必须要有一些同步机制来解决你的操作顺序。一种保护`map`的通用的方法是使用异步读写互斥锁[sync.RWMutex](https://golang.org/pkg/sync/#RWMutex)。

下面的语句声明了一个`counter`变量，它是一个包含`map`和`sync.RWMutex`的匿名数据结构。

    ```go
    var counter = struct{
        sync.RWMutex
        m map[string]int
    }{m: make(map[string]int)}
    ```

执行读操作需要加`read lock`：

    ```go
    counter.RLock()
    n := counter.m["some_key"]
    counter.RUnlock()
    fmt.Println("some_key:", n)
    ```

执行写操作的需要加`write lock`：

    ```go
    counter.Lock()
    counter.m["some_key"]++
    counter.Unlock()
    ```


## 迭代顺序


当用`range`循环迭代一个`map`时，迭代顺序是不明确的且不能保证一次与下一次的顺序完全一致。
如果你需要一个固定的迭代顺序，你必须去单独维护一个存储迭代顺序的数据结构。
下面的例子就用了一个独立的排序后的`slice of keys`来有序打印`map`：

    ```go
    import "sort"

    var m map[int]string
    var keys []int
    for k := range m {
        keys = append(keys, k)
    }
    sort.Ints(keys)
    for _, k := range keys {
        fmt.Println("Key:", k, "Value:", m[k])
    }
    ```
