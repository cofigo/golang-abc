# Go Module使用replace解决fork项目的import问题

## 探究问题所在

如果直接第三方依赖，Golang 是可以满足使用的。但如果我们需要修复第三方依赖的bug，抑或为第三方依赖添加新feature，那将是非常反常识的。

在其他语言中，往往是可以import相对路径的包，因此改动第三方依赖非常简单，只需要fork一下，然后改动相关代码，然后将自己项目的依赖定位到新的地址即可。

但是在 Golang 中不是这样。Golang 中，只允许绝对路径，所有的 import，都将唯一定位到同一个固定的地方，下面举例。

如果我们 fork了我们的一个依赖的仓库，它的结构可能如下：

```shell
forked-repo
    |--a.go # import了some-dir
    |--b.go # import了some-dir
    |--some-dir # 子module
        |--c.go
        |--d.go
```

我们可以将自己的项目的依赖地址定位到 forked-repo 上，这样我们对`a.go`和`b.go`的改动，是可以生效的，这也很符合我们的常识。

但如果我们需要改动`c.go`，那会发现，我们的改动并不会生效；因为`a.go`import 的`some-dir`，地址实际上是`github.com/源地址/xxxx`，我们fork后的地址是`github.com/fork地址/xxx`。这样，子 module 依旧是使用原仓库的，这就是问题所在。



## 解决方案

### 替换依赖地址方式

直接替换所有的依赖地址，可以通过`sed`命令或者其他方法。如果你打算长期依赖于 fork 后的仓库，且需要改动的地方不多的时候，可以考虑这种方式。

但如果你考虑提交 pull request 的话，那这种方式无疑很麻烦。



### 通过Go modules的replace方式

Go modules 的 replace 命令一般而言，是最佳的解决方案。如果你在使用 Go modules，可以考虑这种方式。

官方对 replace 命令的描述：

> replace 指令在顶级 go.mod 中为实际用于满足 go 源文件或 go.mod 文件中的依赖项的内容提供附加控制，而在 building 时忽略除主模块以外的模块中的replace指令。

简而言之就是：

- 在项目的 go.mod 中，使用 replace 命令的话，可以对 import 路径进行替换。也就是可以达到，我们 import 的是a，但在构建的时候，实际使用的是b。
- 可以 replace 为 VCS（github 或者其他地方）或者文件系统路径（可以是绝对路径，也可以是相对于项目根目录的相对路径）。使用文件系统路径这种，在开发调试的使用，非常有用。
- 最顶层 go.mod 的 replace 命令，将影响到自身，以及自身的所有依赖。也就是可以间接改变依赖的项目的import。这样我们在 fork 的项目的 import，不用在 fork 项目里面的 go.mod 进行 replace，直接在使用的项目里replace 即可。



## 使用方式

直接编辑 go.mod：

```go
module nt-pdf-generator

go 1.16

require (
    github.com/xxx/abc v0.2.0 # replace的依赖, 必须要require
)

replace github.com/xxx/abc => ../github.com/hanFengSan/abc # 文件路径不需要附带版本信息
```

然后运行一下：

```shell
$ go mod tidy
```

大功告成。



## 参考

- [go modules的replace使用, 解决fork的项目import问题](https://www.jianshu.com/p/ba900b298ba6)




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))

