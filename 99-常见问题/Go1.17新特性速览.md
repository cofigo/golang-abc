# Go 1.17 新特性速览

按照计划，Go1.17 在 8 月份如期发布了。关于 Go1.17 更多的细节特性，可以参考[官方文档](https://golang.org/doc/go1.17)。

软件包下载方式：

```shell
$ go get golang.org/dl/go1.17
$ go1.17 download
Downloaded   0.0% (    16384 / 135938157 bytes) ...
Downloaded  29.2% ( 39698144 / 135938157 bytes) ...
Downloaded 100.0% (135938157 / 135938157 bytes)
Unpacking /Users/endial/sdk/go1.17/go1.17.darwin-amd64.tar.gz ...
Success. You may now run 'go1.17'
$ go1.17 version
go version go1.17 darwin/amd64
```



## Ports

### Darwin

如 Go 1.16 版本中 [声明](https://golang.org/doc/go1.16#darwin)，Go 1.17 版本需要 macOS 10.13 High Sierra 或更新版本。

### Windows

Go 1.17 在 Windows 平台增加 64-bit ARM 架构支持（ `windows/arm64` port）。同时支持`cgo`。

### OpenBSD

Go 1.17 在 OpenBSD 平台的 64-bit MIPS 架构（`openbsd/mips64` por）增加`cgo`支持。

### ARM64

Go 1.17 在所有系统平台 64-bit ARM 架构提供栈帧（stack frame pointers）支持。之前仅在 Linux、macOS 及 iOS  系统上支持。

### GOARCH 的 loong64 值保留

主编译器暂时还未支持龙芯（ LoongArch）架构，但已经为 `GOARCH`增加保留值`loong64`。这意味着在`GOARCH`启用之前，所有命名为`*_loong64.go`的文件都会被[Go 工具链](https://golang.org/pkg/go/build/#hdr-Build_Constraints)忽略。



## Go语言新特性

Go 1.17 主要新增以下特性：

- [切片转数组指针](https://golang.org/ref/spec#Conversions_from_slice_to_array_pointer): 将切片转换为数组指针，产生指向切片的底层数组的指针。一个类型为`[]T`的表达式`s`可以转换为类型为`*[N]T`的数组指针。如果`a`是转换的结果，那么在有效范围内的索引指向相同的基础元素：`&a[i] == &s[i]` for `0 <= i < N`。如果切片的长度`len(s)`小于数组的长度`N`，则会发生运行时 panic。
- [`unsafe.Add`](https://golang.org/pkg/unsafe#Add): `unsafe.Add(ptr, len)` 将指针 `ptr`偏移`len`并返回新的指针 `unsafe.Pointer(uintptr(ptr) + uintptr(len))`。
- [`unsafe.Slice`](https://golang.org/pkg/unsafe#Slice): 一个类型为`*T`的表达式 `ptr`，`unsafe.Slice(ptr, len)` 将返回一个类型为`[]T`的切片，切片的基础元素起始为 `ptr`，长度与容量都为 `len`。



### 切片与数组转换



#### 数组转切片

先看看在 Go 中如何将数组转为切片（当然，数组指针也是 OK 的）。

一般地，通过 slice 表达式（slice expressions）可以从一个数组得到一个切片。

```
a[low : high : max]
```

其中，max 可以省略。比如：

```
a := [5]int{1, 2, 3, 4, 5}
s := a[1:4]
```

s 就是一个切片。



#### 切片转数组指针

先了解下，为什么会有这样的需求。

该需求来自这个 issue：https://github.com/golang/go/issues/395。rogpeppe 提到，很多时候，函数接收一个 slice 参数，但如果使用数组指针，则允许编译器在编译时检查常量索引。比如这样的情况：

```
func foo(a []int) int {
    return a[0] + a[1] + a[2] + a[3];
}
```

能够编译期进行索引检查。比如这样（当然，最后实现不是这样的）：

```
func foo(a []int) int {
    b := a.[0:4];
    return b[0] + b[1] + b[2] + b[3];
}
```

此外，有时候我们通过数组得到切片，但有时候我们直接创建切片，底层数组是匿名的。如果我们想要获得底层数组怎么办？将切片转为数组指针可以实现这个需求。

看看具体的例子，以下来自 Go 语言规范（针对 Go1.17 这个语言特性新增）：

```
s := make([]byte, 2, 4)
s0 := (*[0]byte)(s)      // s0 != nil
s2 := (*[2]byte)(s)      // &s2[0] == &s[0]
s4 := (*[4]byte)(s)      // panics: len([4]byte) > len(s)

var t []string
t0 := (*[0]string)(t)    // t0 == nil
t1 := (*[1]string)(t)    // panics: len([1]string) > len(s)
```

几个注意的点：

- 当切片的长度小于数组长度（len）时会 panic。所以上面例子中，s4 和 t1 发生了 panic
- 将一个非空切片转为 0 长度的数组，得到的指针不是 nil（如 s0）；但将一个空切片转为 0 长度的数组，得到的指针是 nil（如 t0）；
- 多次转换，并不会创建多个数组（因为得到的是底层数组），这从 `&s2[0] == &s[0]` 可以看出；

所以，总结一下就是，将切片转换为数组指针，产生指向切片的底层数组的指针。如果切片的长度小于数组的长度，则会发生运行时 panic。

不过针对 panic，目前没法做断言检查。只能通过 if 判断了。



#### reflect 注意事项

针对语言这个改动，reflect 包中的 Type 接口有一个方法：ConvertibleTo。之前的说明是这样的：

```
// ConvertibleTo reports whether a value of the type is convertible to type u.
ConvertibleTo(u Type) bool
```

1.17 是这样的：

```
// ConvertibleTo reports whether a value of the type is convertible to type u.
// Even if ConvertibleTo returns true, the conversion may still panic.
// For example, a slice of type []T is convertible to *[N]T,
// but the conversion will panic if its length is less than N.
ConvertibleTo(u Type) bool
```

因为切片转为数组指针可能会 panic，所以才加了这么一句文档说明。

因此，如果通过反射转换做类型转换，虽然通过 ConvertibleTo 判断是可转换的，但调用 Convert 方法依然可能 panic。这点需要特别注意下。



#### 小结

这个语言改变，大部分时候可能用不到。但有些场景可以做到不需要内存拷贝（copy），比如标准库中有一个例子：

```
// https://docs.studygolang.com/src/crypto/sha256/sha256.go?s=5787:5834#L252
func Sum224(data []byte) (sum224 [Size224]byte) {
 var d digest
 d.is224 = true
 d.Reset()
 d.Write(data)
 sum := d.checkSum()
 copy(sum224[:], sum[:Size224])
 return
}
```

官方计划修改为：

```
func Sum224(data []byte) [Size224]byte {
 var d digest
 d.is224 = true
 d.Reset()
 d.Write(data)
 sum := d.checkSum()
 ap := (*[Size224]byte)(sum[:Size224])
 return *ap
}
```



### 新版构建约束



#### 什么是构建约束

构建约束（build constraint），也叫做构建标记（build tag），是在 Go 源文件最开始的注释行，比如：

```
// +build linux
```

看到这个，相信很多人都不陌生，因为这是 Go 一开始就有的特性，在 Go 源码中有很多这样的注释行。上面注释行的意思，这个文件只在 Linux 系统会包含在包中，其他系统会忽略这个文件。

几个注意点：

- 约束可以出现在任何源文件中，比如 `.go`、`.s` 等；
- 必须在文件顶部附近，它的前面只能有空行或其他注释行；可见包子句也在约束之后；
- 约束可以有多行；
- 为了区别约束和包文档，在约束之后必须有空行；

针对某个构建约束，可使用的词如下：

- 特定操作系统，对应 runtime.GOOS 的可用值，比如 linux、windows 等；
- 特定的架构，对应 runtime.GOARCH 的可用值，比如 386、amd64 等；
- 使用的编译器，比如 gc、gccgo；
- 支持 cgo 命令时，可以使用 cgo；
- Go 的主要发布版本，比如 go1.17、go1.16 等；（测试版本和 fixbug 版本不支持）
- 自定义的 tag，编译时通过 `-tags` 传递的值；
- 可以加入任意值，一般用 ignore 来忽略构建；

此外，文件名可以通过 GOOS 和 GOARCH 来做构建约束。



#### 旧版构建约束

从上面看到，构建约束的语法是 `// +build` 这种形式，如果多个条件组合，通过空格、逗号或多行构建约束表示。比如：

```
// +build linux,386
```

你知道什么意思吗？表示在 linux AND 386。逗号表示 AND，空格表示 OR。那看一个复杂的：

```
// +build linux,386 darwin,!cgo
```

是不是有点懵？我也有点懵！它表示的意思是：(linux AND 386) OR (darwin AND (NOT cgo)) 。

有些时候，多个约束分成多行书写，会更易读些：

```
// +build linux darwin
// +build amd64
```

这相当于：(linux OR darwin) AND amd64 。

是不是很复杂，很难记忆？

正因为太复杂，很容易出错。而且，Go 中有不少注释是有特殊意义的，也为了一致性考虑，因此有了新版的构建约束。



#### 新版构建约束

在 Go 源码中，经常会见到类似下面开头的注释：

```
//go:link
```

新版的构建约束，也使用了 `//go:` 开头：

```
//go:build
```

注意 `//` 和 go 之间不能有空格。

同时新版语法使用布尔表达式，而不是逗号、空格等。布尔表达式，会更清晰易懂，出错可能性大大降低。

比如旧语法：

```
// +build linux,386
```

对应的新语法：

```
//go:build linux && 386
```

构建标记的基础语法与其当前形式没有变化，但是构建标记的组合现在是用 Go 的 || 、 && 和 ! 运算符和括号。（请注意，构建标记并不总是有效的 Go 表达式，即使它们共享操作符，因为标记并不总是有效的标识符。例如：”go1.1"。)

新语法可以使用 Go spec 的 EBNF 标记来表示：

```
BuildLine      = "//go:build" Expr
Expr           = OrExpr
OrExpr         = AndExpr   { "||" AndExpr }
AndExpr        = UnaryExpr { "&&" UnaryExpr }
UnaryExpr      = "!" UnaryExpr | "(" Expr ")" | tag
tag            = tag_letter { tag_letter }
tag_letter     = unicode_letter | unicode_digit | "_" | "."
```

采用新语法后，一个文件只能有一行构建语句，而不是像旧版那样有多行。这样可以避免多行的关系到底是什么的问题。

Go1.17 中，gofmt 工具会自动根据旧版语法生成对应的新版语法，为了兼容性，两者都会保留。比如原来是这样的：

```
// +build !windows,!plan9
```

执行 Go1.17 的 gofmt 后，变成了这样：

```
//go:build !windows && !plan9
// +build !windows,!plan9
```

如果文件中已经有了这两种约束形式，gofmt 会根据 `//go:buid` 自动覆盖 `// +build` 的形式，确保两者表示的意思一致。如果只有新版语法，不会自动生成旧版的，这时，你需要注意，它不兼容旧版本了。

另外，Vet 工具现在能够检测出两种语法的不一致。所以，建议大家在编辑器中保存文件时自动执行 gofmt。

早在 Go1.16 时就新增了一个包：go/build/constraint，专门处理新版构建约束。

关于新版约束的设计文档请移步：https://go.googlesource.com/proposal/+/master/design/draft-gobuild.md。



#### 总结

新版本的构建约束可读性更强，更容易书写，不容易出错。有兴趣的可以自己针对构建约束，同时书写两种形式，体会下新版的好处。

最后提醒一点，新版约束中，一定要注意 `//` 和 go 之间不能有空格！



## 标准库的变化

Golang1.17 除了对标准库进行通常的功能更新和修复，还引入了一些重大变化。



### time 包

Unix 时间戳，大家知道单位是什么吗？Java 或 JavaScript 的同学大概率会回答是毫秒，因为这两门语言提供获取“时间戳”的方法，单位是毫秒。但实际上，标准的 Unix 时间戳，单位是秒，标准定义是：

> Unix 时间戳是从 1970 年 1 月 1 日（UTC/GMT 的午夜）开始所经过的秒数，不考虑闰秒。

正因为如此，Go 标准库 time 包提供获取时间戳的方法是 `Unix()`，单位是秒：

```
// Unix returns t as a Unix time, the number of seconds elapsed
// since January 1, 1970 UTC. The result does not depend on the
// location associated with t.
// Unix-like operating systems often record time as a 32-bit
// count of seconds, but since the method here returns a 64-bit
// value it is valid for billions of years into the past or future.
func (t Time) Unix() int64
```

在和客户端/前端协商 API 时，一定要注意时间戳单位的问题。

为了方便，Go1.17 增加加了 Time.UnixMilli 方法，返回 Unix 时间戳的毫秒数，同时也提供了 UnixNano 和 UnixMicro。

此外，如果前端传递一个毫秒的时间戳，可以通过 Go1.17 新的函数 UnixMilli 转为 Time 类型：

```
func UnixMilli(msec int64) Time
```

注意 UnixMilli 函数和 Time.UnixMilli 方法的区别，互逆的关系。



### net/url 包

在这个包中有一个类型 Values，定义如下：

```
type Values map[string][]string
```

它通常用于查询参数和表单值。它提供了 Set、Get、Del 等方法，但没有提供判断某个 key 是否设置了的方法（虽然自己实现不难，但形式不一致），而且这种需求还挺多的。Go1.17 就增加了一个方法：Has，用来判断某个 key 是否设置了。

```
// Has checks whether a given key is set.
func (v Values) Has(key string) bool
```



### net 包

如何判断一个 IP 地址是否是内网地址？你查找标准库会发现没有这样的方法，这时只能自己实现，需要查找 IPv4 标准，看哪些是内网地址，还得处理 IPv6 的情况。

Go1.17 中增加了一个方法：IsPrivate，用来判断一个 IP 地址是否是内网地址：

```
// IsPrivate reports whether ip is a private address, according to
// RFC 1918 (IPv4 addresses) and RFC 4193 (IPv6 addresses).
func (ip IP) IsPrivate() bool
```

是不是方便很多。看它的实现，自己实现可能不那么容易：

```
func (ip IP) IsPrivate() bool {
 if ip4 := ip.To4(); ip4 != nil {
  // Following RFC 1918, Section 3. Private Address Space which says:
  //   The Internet Assigned Numbers Authority (IANA) has reserved the
  //   following three blocks of the IP address space for private internets:
  //     10.0.0.0        -   10.255.255.255  (10/8 prefix)
  //     172.16.0.0      -   172.31.255.255  (172.16/12 prefix)
  //     192.168.0.0     -   192.168.255.255 (192.168/16 prefix)
  return ip4[0] == 10 ||
   (ip4[0] == 172 && ip4[1]&0xf0 == 16) ||
   (ip4[0] == 192 && ip4[1] == 168)
 }
 // Following RFC 4193, Section 8. IANA Considerations which says:
 //   The IANA has assigned the FC00::/7 prefix to "Unique Local Unicast".
 return len(ip) == IPv6len && ip[0]&0xfe == 0xfc
}
```



### 其他包

math 包新提供了 MaxInt、MinInt、MaxUint 三个常量，分别对应 int 的最大、最小值和 uint 的最大值，这样我们不需要自己判断当前 CPU 架构确定最大最小值。

io/fs 包新增加 FileInfoToDirEntry 函数，它用于获取一个 FileInfo 的 DirEntry 信息，在操作文件系统时可能会用到。

```
func FileInfoToDirEntry(info FileInfo) DirEntry
```

database/sql 包新增加了 NullByte 和 NullInt16，用于表示可能为 null 的 int16 和 byte 类型（我个人不建议创建数据表时支持 null，这样处理起来比较麻烦，建议全部 NOT NULL）。



## 其他改进

更多信息可以查看[golang1.17 release notes](https://golang.org/doc/go1.17)




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))
