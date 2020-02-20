# GoLang Modules

Go Modules 是 golang 1.11 才新增加的一个特性，虽然还处于试验阶段，不过官方说明中表示后续的版本会持续保持对已发布功能的兼容，所以我们还是可以大胆地进行尝试……



## 快速上手

在 $GOPATH 路径之外创建一个目录，并初始化一个新的 module：

```shell
$ make -p ~/project/hello$ cd ~/project/hello$ go mod init hello
```



这样子就会生成一个 go.mod 文件；生成 main.go 文件并写一段简单的代码：

```go
package main 

import (
  "fmt"
  "rsc.io/quote"
)

func main() {
  fmt.Println(quote.Hello())
}
```



执行 go build，生成可执行文件 hello，以及 go.sum 文件：

```shell
$ go build

$ ./hello
Hello, world.

$ ls -al
-rw-rw-r-- 1 jachua jachua   42 1月 22 10:13 go.mod
-rw-rw-r-- 1 jachua jachua   499 1月 22 10:08 go.sum
-rwxrwxr-x 1 jachua jachua 2223953 1月 22 10:13 hello
-rw-rw-r-- 1 jachua jachua   94 1月 22 10:08 main.go
```



 查看 go.mod 和 go.sum 文件的内容，可以发现其中包含了依赖的包：

```shell
$ cat go.mod
module hello

require rsc.io/quote v1.5.2

$ cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
```



当我们执行 go build 或者 go get 命令时，项目所需要的依赖将会自动下载到 $GOPATH/pkg/mod/ 目录下（作为缓存，可被其他项目依赖使用），并更新 go.mod 和 go.sum。

当然，我们还可以指定特定的版本或者分支比如： go get foo@1.2.3 或 go get foo@master。此外，还有一些**常用的命令**：

- go list -m all：列出所有依赖的最终版本
- go list -u -m all：列出可更新的依赖
- go get -u：更新依赖
- go build ./… 或 go test ./… ：在项目根目录下执行，构建/测试 module 中 的包
- go mod download：下载依赖到本地缓存
- go mod tidy：增加需要的依赖，删除不需要的依赖
- go mod vendor：把依赖复制到项目的 vendor 目录



## 基本概念

### Modules

*A module is a collection of related Go packages that are versioned together as a single unit.*

- 一个 repository 包含一个或多个 Go modules；
- 每个 module 包含一个或多个 Go packages；
- 每个 package 由单一目录下的一个或多个 Go 源文件 组成；

### go.mod

一个 Module 由位于根目录的 go.mod 和一些 go 文件构成，module 可以放在 $GOPATH 目录之外。



## 如何使用

### 初始化

首先，需要安装 Go 1.11 以上版本，然后：

- 在 $GOPATH/src 目录外，直接执行 go 命令（环境变量GO111MODULE无需设置，或设置为 auto）；
- 在 $GOPATH/src 目录下，设置 GO111MODULE=on，再执行 go 命令；

如果是在 $GOPATH/src 目录下，执行 go mod init 将会自动确定 module 的名字，不然就需要输入 module 名字 go mod init moduleName。



### 更新/回退

使用 go get 命令：

- 执行 go get -u 更新最新版本（到 minor 或者 patch 版本）
- 为指定的包选择指定的版本：
  - go get foo 相当于 go get foo@latest 
  - go get foo@1.2.3 
  - go get foo@e3702bed2
  - go get foo@'<1.6.2′

### 发布

大部分的步骤将会被未来的 go release 工具自动化执行，不过目前我们可以这么操作：

- 执行 go mod tidy 移除不必要的依赖，因为 go build/test 并不会自动删除不再需要的依赖；
- 执行 go test all 测试；
- 把 go.sum 和 go.mod 加入到 vcs 中提交；



### Vendor

前面我们提到了，如果启用了 Modules （$GOPATH/src 外或 GO111MODULE=on），默认的依赖都是下载到 $GOPATH/pkg/mod 目录下，我们也可以使用 go mod vendor 将依赖添加到项目根目录的 vendor 目录中。

不过，此时如果我们直接执行 go build 命令，那么使用仍然是 go.mod 描述的依赖（ $GOPATH/pkg/mod ）而非 vendor 中的依赖，我们需要指定 -mod=vendor 参数：

```shell
$ go build -mod=vendor
```



这样子 go 命令就可以忽略了 go.mod，而认为 vendor 目录包含了正确的依赖。

另外，如果指定了 -mod=readonly 参数，则 go build 过程中 go.mod 将不会被自动更新（如果 go.mod 需要被修改，那么构建就是失败）。这个参数常用于 CI 自动化构建，保证正确的依赖关系。

```shell
$ go build -mod=readonly
```



> **最后，注意 -mod 构建参数只对 go build 有效，而 go test 会将其忽略。**

 

## 参考：

- https://github.com/golang/go/wiki/Modules
- https://colobu.com/2018/08/27/learn-go-module/
- https://roberto.selbach.ca/intro-to-go-modules/
- https://www.youtube.com/watch?v=F8nrpe0XWRg&list=PLq2Nv-Sh8EbbIjQgDzapOFeVfv5bGOoPE&index=3&t=0s
- ChenJiehua的[《Golang Modules 笔记》](https://chenjiehua.me/golang/go-modules-intro.html)

