# GoLang构建模式

Golang的构建模式（buildmode）指的是编译器如何编译源码构建出相关的对象文件，最常见的情况下就是生成一个可执行的二进制文件。

在 `go build` 和 `go install` 命令中，我们可以指定 `-buildmode` 参数来让编译器构建出特定的对象文件。



## buildmode 一览

通过命令 `go help buildmode`，可以看到其支持的选项：

```shell
-buildmode=archive
-buildmode=c-archive
-buildmode=c-shared
-buildmode=default
-buildmode=shared
-buildmode=exe
-buildmode=pie
-buildmode=plugin
```

`default`为最常用的默认模式，在没有指定构建模式时，默认为使用`default`模式。



## C静态链接库：c-archive

### **说明：**

> *Build the listed main package, plus all packages it imports, into a C archive file. The only callable symbols will be those functions exported using a cgo //export comment. Requires exactly one main package to be listed.*

c-archive 也就是将package main 中导出的方法（// export 标记）编译成 .a 文件，这样其它 c 程序就可以静态链接该文件，并调用其中的方法。



### Go示例：

首先写一个简单 add.go ：

```go
package main 

import "fmt"
import "C" 

func main(){} 

//export Add
func Add(a, b int) int{
  fmt.Printf("%d + %d = %d\n", a, b, a+ b)
  return a+b
}
```



> - 需要输出给外部使用的函数，添加`//export xxxx`说明
> - 需要一个main package，包含一个空的`main()`函数



然后使用 `-buildmode=c-archive` 编译：

```shell
$ go build -buildmode=c-archive add.go
```



生成两个文件： add.a 和 add.h，查看一下文件的类型：

```shell
$ file add.a add.h
add.a: current ar archive random library
add.h: c program text, ASCII text
```



 在 add.h 中我们看到 Add() 函数的定义：

```c++
#ifdef __cplusplus
extern "C" {
#endif 

extern GoInt Add(GoInt p0, GoInt p1); 
  
#ifdef __cplusplus
}
#endif
```



### C示例：

接下来，我们再写一个简单的 C 程序：

```c
# include "add.h" 

int main(void) {
  Add(1, 2);
  return 0;
}
```



加上 add.a 文件编译一下：

```shell
$ cc myadd.c add.a
```



生成一个可执行文件 a.out，执行 ./a.out：

```shell
$ ./a.out
1 + 2 = 3
```





## C 动态链接库：c-shared

### 说明：

> *Build the listed main package, plus all packages it imports, into a C shared library. The only callable symbols will be those functions exported using a cgo //export comment. Requires exactly one main package to be listed.*

c-shared 也就是将 package main 中导出的方法（// export 标记）编译成一个动态链接库（.so 或 .dll 文件），这样其它 c 程序就可以调用其中的方法。



### Go示例：

继续使用上面的 add.go，改用 `-buildmode=c-shared` 来编译：

```shell
$ go build -buildmode=c-shared -o add.so add.go
```



这次生成了 add.so 和 add.h 两个文件：

```shell
$ file add.so add.h
add.so: Mach-O 64-bit dynamically linked shared library x86_64
add.h: c program text, ASCII text
```



### C示例：

继续使用上面的 myadd.c进行编译：

```shell
$ cc myadd.c add.so
```



同样是生成了 a.out 可执行文件，但是留意一下前后两次的体积：

- 使用 add.so 编译生成的 a.out 才 8.2KB；
- 使用 add.a 编译生成的 a.out 有 1.8MB；

如果我们将 add.so 删除，再运行 a.out：

```shell
dyld: Library not loaded: add.so
	Referenced from: /Users/jachua/go/src/buildmode/c-archive/./a.out 
	Reason: image not found
[1]  94945 abort   ./a.out
```



另外，我们也可以指定环境变量 `LD_LIBRARY_PATH` 告诉操作系统到哪里去寻找动态链接库：

```shell
// Linux
$ LD_LIBRARY_PATH=./lib ./out

// macOS
$ DYLD_LIBRARY_PATH=./lib ./out
```



### 版本要求：

- G0 1.5

> *For the amd64 architecture only, the compiler has a new option, `-dynlink`, that assists dynamic linking by supporting references to Go symbols defined in external shared libraries.*

- G0 1.6

> *The implementation of [build modes started in Go 1.5](https://golang.org/doc/go1.5#link) has been expanded to more systems. This release adds support for the `c-shared` mode on `android/386`, `android/amd64`, `android/arm64`, `linux/386`, and `linux/arm64`; for the `shared` mode on `linux/386`, `linux/arm`, `linux/amd64`, and `linux/ppc64le`; and for the new `pie` mode (generating position-independent executables) on `android/386`, `android/amd64`, `android/arm`, `android/arm64`, `linux/386`, `linux/amd64`, `linux/arm`, `linux/arm64`, and `linux/ppc64le`. See the [design document](https://golang.org/s/execmodes) for details.*

- Go 1.10

> *The various [build modes](https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/edit) have been ported to more systems. Specifically, `c-shared` now works on `linux/ppc64le`, `windows/386`, and `windows/amd64`; `pie` now works on `darwin/amd64` and also forces the use of external linking on all systems; and `plugin` now works on `linux/ppc64le` and `darwin/amd64`.*

- Go 1.11

> *The build modes `c-shared` and `c-archive` are now supported on `freebsd/amd64`.*





## Go动态链接库：shared

### 说明：

> *Combine all the listed non-main packages into a single shared library that will be used when building with the -linkshared option. Packages named main are ignored.*

shared 与 c-shared 类似，不过它是用来给 golang 构建动态链接库的。它将 非main 的package 编译为动态链接库，并在构建其他 go程序时使用 -linkshared 参数指定。



### 示例：

首先我们写一个非常简单的 hello.go：

```go
package main 

import "fmt" 

func main(){
  fmt.Println("hello world")
}
```



然后，我们需要将 golang 的所有标准库 std 编译安装为 shared：

```shell
// -buildmode=shared 暂不支持 macOS
$ go install -buildmode=shared std
```



接着再用 -linkshared 编译 hello.go：

```shell
$ go build -linkshared hello.go
```



可以看到生成的可执行文件体积才 20KB ，相比正常的 go build hello.go 生成的 1.9MB 小非常多。我们可以使用 ldd 命令来查看调用链：

```shell
$ ldd hello
	linux-vdso.so.1 (0x00007ffcb0db9000)
  libstd.so => /usr/local/go/pkg/linux_amd64_dynlink/libstd.so (0x00007f2d5c1cb000)
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2d5bdda000)
  libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f2d5bbd6000)
  libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f2d5b9b7000)
  /lib64/ld-linux-x86-64.so.2 (0x00007f2d5eb96000)
```



当然如果缺少了其中某个链接库或者版本不匹配，都将导致无法正常运行，所以一般情况下这种构建模式很少使用。





## Go插件：plugin

### 说明：

> *Build the listed main packages, plus all packages that they* *import, into a Go plugin. Packages not named main are ignored.*

plugin 模式是 golang 1.8 才推出的一个特殊的构建方式，它将 package main 编译为一个 go 插件，并可在运行时动态加载。



### 示例：

首先，我们来写一个简单的 greeting，生成并编辑文件`greeter.go`：

```go
package main 

import "fmt" 

type greeting string 

func (g greeting) Greet() {	
  fmt.Println("hello world")
} 

var Greeter greeting
```



然后将其编译为一个 go 插件：

```shell
$ go build -buildmode=plugin -o greeter.so greeter.go
```



本次编译生成 greeter.so 文件：

```shell
$ file greeter.so
greeter.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=f5729749535c706a6095994f9510da92ccc8f9c6, not stripped
```



再使用 golang 官方的 plugin 库来调用这个插件，生层并编辑 main.go 文件：

```go
package main

import (
  "fmt"
  "os"
  "plugin"
)

type Greeter interface {
  Greet()
}

func main() {
  plug, err := plugin.Open("./greet/greeter.so")
  if err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
  
  symGreeter, err := plug.Lookup("Greeter")	
  if err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
  
  var greeter Greeter
  greeter, ok := symGreeter.(Greeter)
  if !ok {
    fmt.Println(err)
    os.Exit(1)
  }
  greeter.Greet()
}
```



### 版本要求

Go 1.8

> *Go now provides early support for plugins with a “`plugin`” build mode for generating plugins written in Go, and a new [`plugin`](https://golang.org/pkg/plugin/) package for loading such plugins at run time. Plugin support is currently only available on Linux. Please report any issues.*

G0 1.10

> *The various [build modes](https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/edit) have been ported to more systems. Specifically, `c-shared` now works on `linux/ppc64le`, `windows/386`, and `windows/amd64`; `pie` now works on `darwin/amd64` and also forces the use of external linking on all systems; and `plugin` now works on `linux/ppc64le` and `darwin/amd64`.*

 

## 参考

- *https://medium.com/learning-the-go-programming-language/calling-go-functions-from-other-languages-4c7d8bcc69bf*
- *https://docs.google.com/document/d/1nr-TQHw_er6GOQRsF6T43GGhFDelrAP0NqSS_00RgZQ/edit?pli=1#*
- *https://blog.csdn.net/linuxandroidwince/article/details/78723441*
- *https://blog.lab99.org/post/golang-2017-10-01-video-go-build-mode.html*
- *https://medium.com/learning-the-go-programming-language/writing-modular-go-programs-with-plugins-ec46381ee1a9*
- ChenJiehua的[《Golang的构建模式》](https://chenjiehua.me/golang/golang-buildmode.html)



----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))
