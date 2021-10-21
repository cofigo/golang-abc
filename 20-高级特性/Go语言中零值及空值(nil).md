# Go语言空值/零值(nil)

在 Go 语言中，布尔类型的零值（初始值）为 false，数值类型的零值为 0，字符串类型的零值为空字符串`""`，而指针、切片、映射、通道、函数和接口的零值则是 nil。

nil 是 Go 语言中一个预定义好的标识符，有过其他编程语言开发经验的开发者也许会把 nil 看作其他语言中的 null（NULL），其实这并不是完全正确的，因为 Go 语言中的 nil 和其他语言中的 null 有很多不同点。

下面通过几个方面来介绍一下 Go 语言中 nil。



## nil 标识符是不能比较的

```go
package main
import (
    "fmt"
)
func main() {
    fmt.Println(nil==nil)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
# command-line-arguments
.\main.go:8:21: invalid operation: nil == nil (operator == not defined on nil)
```



这点和 python 等动态语言是不同的，在 python 中，两个 None 值永远相等。

```shell
>>> None == None
True
```

从上面的运行结果不难看出，`==`对于 nil 来说是一种未定义的操作。



## nil 不是关键字或保留字

nil 并不是Go语言的关键字或者保留字，也就是说我们可以定义一个名称为 nil 的变量，比如下面这样：

```go
var nil = errors.New("my god")
```

虽然上面的声明语句可以通过编译，但是并不提倡这么做。



## nil 没有默认类型

```go
package main
import (
    "fmt"
)
func main() {
    fmt.Printf("%T", nil)
    print(nil)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
# command-line-arguments
.\main.go:9:10: use of untyped nil
```



## 不同类型 nil 的指针是一样的

```go
package main
import (
    "fmt"
)
func main() {
    var arr []int
    var num *int
    fmt.Printf("%p\n", arr)
    fmt.Printf("%p", num)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
0x0
0x0
```

通过运行结果可以看出 arr 和 num 的指针都是 0x0。



## 不同类型的 nil 是不能比较的

```go
package main
import (
    "fmt"
)
func main() {
    var m map[int]string
    var ptr *int
    fmt.Printf(m == ptr)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
# command-line-arguments
.\main.go:10:20: invalid operation: arr == ptr (mismatched types []int and *int)
```



## 两个相同类型的 nil 值也可能无法比较

在Go语言中 map、slice 和 function 类型的 nil 值不能比较，比较两个无法比较类型的值是非法的，下面的语句无法编译。

```go
package main
import (
    "fmt"
)
func main() {
    var s1 []int
    var s2 []int
    fmt.Printf(s1 == s2)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
# command-line-arguments
.\main.go:10:19: invalid operation: s1 == s2 (slice can only be compared to nil)
```

通过上面的错误提示可以看出，能够将上述不可比较类型的空值直接与 nil 标识符进行比较，如下所示：

```go
package main
import (
    "fmt"
)
func main() {
    var s1 []int
    fmt.Println(s1 == nil)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
true
```



## nil 是 map、slice、pointer、channel、func、interface 的零值

```go
package main
import (
    "fmt"
)
func main() {
    var m map[int]string
    var ptr *int
    var c chan int
    var sl []int
    var f func()
    var i interface{}
    fmt.Printf("%#v\n", m)
    fmt.Printf("%#v\n", ptr)
    fmt.Printf("%#v\n", c)
    fmt.Printf("%#v\n", sl)
    fmt.Printf("%#v\n", f)
    fmt.Printf("%#v\n", i)
}
```

运行结果如下所示：

```shell
$ go run .\main.go
map[int]string(nil)
(*int)(nil)
(chan int)(nil)
[]int(nil)
(func())(nil)
<nil>
```

零值是Go语言中变量在声明之后但是未初始化被赋予的该类型的一个默认值。



## 不同类型的 nil 值占用的内存大小可能是不一样的

一个类型的所有的值的内存布局都是一样的，nil 也不例外，nil 的大小与同类型中的非 nil 类型的大小是一样的。但是不同类型的 nil 值的大小可能不同。

```go
package main
import (
    "fmt"
    "unsafe"
)

func main() {
    var p *struct{}
    fmt.Println( unsafe.Sizeof( p ) ) // 8
    var s []int
    fmt.Println( unsafe.Sizeof( s ) ) // 24
    var m map[int]bool
    fmt.Println( unsafe.Sizeof( m ) ) // 8
    var c chan string
    fmt.Println( unsafe.Sizeof( c ) ) // 8
    var f func()
    fmt.Println( unsafe.Sizeof( f ) ) // 8
    var i interface{}
    fmt.Println( unsafe.Sizeof( i ) ) // 16
}
```

运行结果如下所示：

```shell
$ go run .\main.go
8
24
8
8
8
16
```

具体的大小取决于编译器和架构，上面打印的结果是在 64 位架构和标准编译器下完成的，对应 32 位的架构的，打印的大小将减半。




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))