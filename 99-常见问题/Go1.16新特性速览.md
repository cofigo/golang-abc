# Go 1.16 新特性速览



## 语言內建的资源嵌入支持

之前市面上已经有很多把今天文件嵌入golang二进制程序的工具了，这次 Golang 官方将这一功能加入了`embed`标准库，从语言层面上提供了支持。

embed 的使用教程，可参照[Go语言中embed内嵌静态资源简明教程](../03-高级特性/Go语言中embed内嵌静态资源简明教程.md)；或参照官方[推荐教程](https://blog.carlmjohnson.net/post/2021/how-to-use-go-embed/)。



## 支持arm64

m1 芯片可谓是最近的焦点，Golang 自然也不会落下。

在 Golang1.16 中官方已经支持`darwin/arm64`平台，cgo 和编译成 c 语言可调用的动态/静态链接库的功能也已支持。同样受益的还有 bsd 家族的 arm64 版本。

现在可以在新版 mac 上尝试 Golang 了。

不过 plugin 模式的支持仍在进行中，想要完整支持 arm64 还需要一段时间。



## go modules的新特性

本次更新依旧带来了许多 modules 的新特性。

### GO111MODULE现在默认为on

Golang 1.16开始默认启用 modules，这在 1.15 的时候已经预告过了。现在 GO111MODULE 的默认值为 on。

不过 Golang 还是提供了一个版本的适应期，如果你还不习惯 modules，可以把 GO111MODULE 设置回 auto。在1.17 中这个环境变量将会被删除。

都1202年了，也该学学 go modules 怎么用了。



### go build不再更改mod相关文件

go build 会自动下载依赖，这会更新 mod 文件。

现在这一行为被禁止了。想要安装、更新依赖只能使用 go get 命令，go build 和 go test 将不会再做这类工作。



### go install的变化

go install 在 1.16 中也有了不小的变化。

首先是通过 go install my.module/tool@1.0.0 这样在 module 末尾加上版本号，可以在不影响当前 mod 的依赖的情况下安装 Golang 程序。

go install 是未来唯一可以安装 Golang 程序的命令，go get 的编译安装功能现在可以靠`-d`选项关闭，而未来编译安装功能会从 go get 移除。

也就是说 Golang 的命令各司其职，不再长臂管辖了。



### 新的GOVCS环境变量

新的 GOVCS 环境变量指定了 Golang 用什么版本控制工具下载源代码。

其格式为：`GOVCS=<module prefix>:<tool name>,[<module prefix>:<tool name>, ...]`

其中 module prefix 为 github.com 等，而 tool name 就是版本控制工具的名字，比如 git，svn。

一个更具体的例子是：`GOVCS=github.com:git,evil.com:off,*:git|hg`

module prefix 也可以用`*`通配任何模块的前缀。

tool name 还可以设置为 all 和 off，all 代表允许使用任何可用的工具，而off则表示不允许使用任何版本控制工具。

不过现在设置为 off 的模块的代码仍然可能会被下载。

更多的细节可以参考`go help vcs`。



### 相对路径导入不在被允许

Golang1.16 开始禁止 import 导入的模块以`.`开头，模块路径中也不允许出现任何非 ASCII 字符，所以下面的代码不再合法：

```golang
import (
    "./tools/factory"
    "../models/user"
    "some.pkg.com/杀马特/音乐工厂"
)
```

对非 ASCII 字符一如既往的不友好，不过也只能按规矩办事了。



## 标准库的变化

Golang1.16 除了对标准库进行通常的功能更新和修复，还引入了一些重大变化。

### testing

testing 包主要的变化是在测试用例里调用`os.Exit(0)`会从程序终止变成测试失败。

比如这个：

```golang
package main

import (
    "os"
    "testing"
)

func TestXXX(t *testing.T) {
    t.Log("exit")
    os.Exit(0)
}
```

现在会是这样的输出：

```bash
$ go test -v a_test.go

=== RUN   TestXXX
    a_test.go:9: exit
--- FAIL: TestXXX (0.00s)
panic: unexpected call to os.Exit(0) during test [recovered]
        panic: unexpected call to os.Exit(0) during test

goroutine 18 [running]:
testing.tRunner.func1.2(0x51b920, 0x56cc28)
        /usr/local/go/src/testing/testing.go:1144 +0x332
testing.tRunner.func1(0xc000082600)
        /usr/local/go/src/testing/testing.go:1147 +0x4b6
panic(0x51b920, 0x56cc28)
        /usr/local/go/src/runtime/panic.go:965 +0x1b9
os.Exit(0x0)
        /usr/local/go/src/os/proc.go:68 +0x6d
command-line-arguments.TestXXX(0xc000082600)
        /tmp/a_test.go:10 +0x76
testing.tRunner(0xc000082600, 0x54df18)
        /usr/local/go/src/testing/testing.go:1194 +0xef
created by testing.(*T).Run
        /usr/local/go/src/testing/testing.go:1239 +0x2b3
FAIL    command-line-arguments  0.004s
FAIL
```



### ioutils包已经废弃

Golang1.16 已经标记`io/ioutil`为废弃，函数被转移到了 os 和 io 这两个包里，具体见下表：

| ioutil旧函数 | 新函数        |
| ------------ | ------------- |
| Discard      | io.Discard    |
| NopCloser    | io.NopCloser  |
| ReadAll      | io.ReadAll    |
| ReadDir      | os.ReadDir    |
| ReadFile     | os.ReadFile   |
| WriteFile    | os.WriteFile  |
| TempDir      | os.MkdirTemp  |
| TempFile     | os.CreateTemp |

现在开始可以做移植了。



### tcp半连接队列扩容

在 Linux kernel 4.1 以前，Golang 设置 tcp 的 listen 队列的长度是从 /proc/sys/net/core/somaxconn 获取的，通常为4096。

而在 4.1 以后 Golang 会直接设置半连接队列的长度为`2^32 - 1`也就是 4294967295。

更大的半连接队列意味着可以同时处理更多的新加入请求，而且不用再读取配置文件性能也会略微提升。



### 重大更新io/fs

Golang1.16 除了支持嵌入静态资源外，最大的变化就是引入了 io/fs 包。

Golang 认为文件的 io 操作是依赖于文件系统（filesystem，fs）的，所以决定模仿 Linux 的 vfs 做一套基于 fs 的 io 接口。

这样做的目的有三个：

1. os 包应该专注于和系统交互而不是包含一部分 io 接口
2. io 包和 os 包分别包含了 io 接口的一部分，导致互相依赖职责不清晰
3. 可以把有关联的一部分文件或者数据组成虚拟文件系统，供通用接口处理提升程序的可扩展性，比如 zip 打包的文件

所以 io/fs 诞生了。

fs 包中主要包含了下面几种数据类型（都是接口类型）：

| 名称        | 作用                                                   |
| ----------- | ------------------------------------------------------ |
| FS          | 文件系统的抽象，有一个Open方法用来从FS打开获取文件数据 |
| DirEntry    | 描述目录项目（包含目录自身）的数据结构                 |
| File        | 描述文件数据的结构，包含Stat，Read，Close方法          |
| ReadDirFile | 在File的基础上支持ReadDir，可以代表目录自身            |
| FileMode    | 描述文件类型，比如是通常文件还是套接字或者是管道       |
| FileInfo    | 文件的元数据，例如创建时间等                           |

其中有一些接口和 os 包中的同名，实际上是 os 包引入 fs 包后起的别名。

对于 FS，还有以下的扩展，以便增量描述文件系统允许的操作：

| 名称       | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| GlobFS     | 增加Glob方法，可以用通配符查找文件                           |
| ReadDirFS  | 增加ReadDir方法，可以遍历目录                                |
| ReadFileFS | 增加ReadFile方法，可以用文件名读取文件所有内容               |
| StatFS     | 增加Stat方法，可以获得文件/目录的元信息                      |
| SubFS      | 增加Sub方法，Sub方法接受一个文件/目录的名字，从这个名字作为根目录返回一个新的文件系统对象 |

fs 包还提供了诸如 Glob，WalkDir 等传统的文件操作接口。

fs 的主要威力在于处理 zip、tar 文件，以及 http 的文件接口时可以大幅简化代码。而且新的`embed`静态资源嵌入也是依赖 fs 实现的。

因为只是速览的缘故，无法详尽介绍 io/fs 包，你可以参考 Golang 的文档或[这篇文章](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/)做进一步了解。



## 其他改进

其他的改进包括 Unicode 更新到了 13.0、新增加了 runtime/metrics 包已提供更好更规范的运行时信息等。

同时 1.16 优化了链接器，现在它在 linux/amd64 上比 1.15 快了 20-25%，内存占用减少了 5-15%。

在 Windows 上已经全面支持了地址空间布局随机化（ASLR），此前不支持将 Golang 编译为 dll 时启用 ASLR。

本次更新中语言本身没有什么变化。

更多信息可以查看[golang1.16 release notes](https://golang.org/doc/go1.16)




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))
