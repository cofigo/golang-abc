# Go 语言中的下划线"`_`"用法

## 忽略返回值
这个应该是最简单的用途，比如某个函数返回三个参数，但是我们只需要其中的两个，另外一个参数可以忽略，这样的话代码可以这样写：

```go
v1, v2, _ := function(...)
```



## 用在变量(特别是接口断言)
例如我们定义了一个接口(interface)：

type Foo interface {
     Say()
}
1
2
3
然后定义了一个结构体(struct)

type Dog struct {

}
1
2
3
然后我们希望在代码中判断Dog这个struct是否实现了Foo这个interface

var _ Foo = Dog{}
1
上面用来判断Dog是否实现了Foo, 用作类型断言，如果Dog没有实现Foo，则会报编译错误

## 用在import package
假设我们在代码的import中这样引入package：

import _ "test/foo"
1
这表示呢在执行本段代码之前会先调用test/foo中的初始化函数(init)，这种使用方式仅让导入的包做初始化，而不使用包中其他功能

例如我们定义了一个Foo struct，然后对它进行初始化

package foo
import "fmt"
type Foo struct {
        Id   int
        Name string
}
func init() {
        f := &Foo{Id: 123, Name: "abc"}
        fmt.Printf("init foo object: %v\n", f)
}
1
2
3
4
5
6
7
8
9
10
然后在main函数里面引入test/foo

package main
import (
        "fmt"
        _ "test/foo"
)
func main() {
        fmt.Printf("hello world\n")
}
1
2
3
4
5
6
7
8
运行结果如下

init foo object: &{123 abc}
hello world
1
2
我们可以看到：在main函数输出”hello world”之前就已经对foo对象进行初始化了！

## 参考

- [golang中的下划线(_)用法](https://blog.csdn.net/raoxiaoya/article/details/108872221)