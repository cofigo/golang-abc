# Go 语言中 embed 内嵌静态资源简明教程



## embed简介

### embed是什么

embed 是在 Go 1.16 中新加包。它通过***`//go:embed`***指令，可以在编译阶段将静态资源文件打包进编译好的程序中，并提供访问这些文件的能力。



### 为什么需要embed包

- **部署过程更简单。**传统部署要么需要将静态资源与已编译程序打包在一起上传，或者使用 docker 和 dockerfile 自动化前者，这在精神上是很麻烦的。
- **确保程序的完整性。**在运行过程中损坏或丢失静态资源通常会影响程序的正常运行。
- **您可以独立控制程序所需的静态资源。**

最常见的方法（例如静态网站的后端程序）要求将程序连同其所依赖的 html 模板、css、js 和图片以及静态资源的路径一起上传到生产服务器。必须正确配置 Web 服务器，以便用户访问它。

现在，我们将所有这些资源都嵌入到程序中。我们只需要部署一个二进制文件并为程序本身配置它们即可。部署过程已大大简化。



### embed的常用场景

以下列举一些静态资源文件需要被嵌入到程序中的常用场景：

- **Go模板：**模板文件必须可用于二进制文件（模板文件需要对二进制文件可用）。 对于 Web 服务器二进制文件或那些通过提供 init 命令的 CLI 应用程序，这是一个相当常见的用例。 在没有嵌入的情况下，模板通常内联在代码中。例如示例 qbec init 的 init 命令：[https://qbec.io/userguide/tour/#initialize-a-new-qbec-app](https://link.zhihu.com/?target=https%3A//qbec.io/userguide/tour/%23initialize-a-new-qbec-app)*
- **静态web服务：**有时，静态文件（如 index.html 或其他 HTML、JavaScript 和 CSS 文件之类的静态文件）需要使用 Golang 服务器二进制文件进行传输，以便用户可以运行服务器并访问这些文件。例如示例 web server 中嵌入静态资源文件：[https://github.com/gobuffalo/toodo/tree/master/assets](https://link.zhihu.com/?target=https%3A//github.com/gobuffalo/toodo/tree/master/assets)
- **数据库迁移：**另一个使用场景是通过嵌入文件被用于数据库迁移脚本。参考示例数据库迁移文件：[https://github.com/bigpanther/t...](https://link.zhihu.com/?target=https%3A//github.com/bigpanther/trober/tree/786dc471ea0d9b4a9e934d7e3c192de214f7c173/migrations)



## embed的基本使用

embed 包是 Golang1.16 中的新特性，所以，请确保你的 Golang 环境已经升级到了 1.16 版本。 



### embed的基本语法

基本语法非常简单，首先导入***embed包，***然后使用指令***//go:embed*** 文件名 将对应的文件或目录结构导入到对应的变量上。 例如： 在当前目录下新建文件 version.txt，文件内容： 0.0.1

```go
package main

import (
    _ "embed"
    "fmt"
)

//go:embed version.txt
var version string		// 读取静态资源文件中内容，并赋值给字符串变量 version

func main() {
    fmt.Printf("version: %q\n", version)
}
```



### embed的数据类型

embed一共有三种数据格式：

| 数据类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| []byte   | 表示数据存储为二进制格式，如果只使用[]byte和string需要以import (_ "embed")的形式引入embed标准库 |
| string   | 表示数据被编码成utf8编码的字符串，因此不要用这个格式嵌入二进制文件比如图片，引入embed的规则同[]byte |
| embed.FS | 表示存储多个文件和目录的结构，[]byte和string只能存储单个文件 |



### 嵌入为字符串类型

将文件内容嵌入到字符串变量中。例如： 在当前目录下新建文件 version.txt，文件内容： 0.0.1

```go
package main
import (
    _ "embed"
    "fmt"
)

//go:embed version.txt
var version string

func main() {
    fmt.Printf("version %q\n", version)
}
```



### 嵌入为字节数组类型

将文件内容嵌入到字节数组变量（slice）中。例如： 在当前目录下新建文件 version.txt，文件内容： 0.0.1

```go
package main
import (
    _ "embed"
    "fmt"
)

//go:embed version.txt
var versionByte []byte

func main() {
    fmt.Printf("version %q\n", string(versionByte))
}
```



### 嵌入为文件类型

可以嵌入为一个文件系统，这在嵌入多个文件的时候非常有用。

#### 嵌入一个文件

```go
package main
import (
	"embed"
	"fmt"
)

//go:embed hello.txt
var f embed.FS

func main() {
	data, _ := f.ReadFile("hello.txt")
	fmt.Println(string(data))
}
```



#### 嵌入多个文件

```go
package main
import (
	"embed"
	"fmt"
)

//go:embed hello.txt hello2.txt
var f embed.FS

func main() {
	data, _ := f.ReadFile("hello.txt")
	fmt.Println(string(data))
	data, _ = f.ReadFile("hello2.txt")
	fmt.Println(string(data))
}
```

上述嵌入命令也可以分割为多行：

```go
//go:embed hello.txt
//go:embed hello2.txt
var f embed.FS
```

重复的`go:embed`指令嵌入为 embed.FS 是支持的，相当于一个。



#### 嵌入一个目录

将文件目录结构映射成 embed.FS 文件类型。例如：在当前文件夹下存在目录及文件 static/version.txt

```go
package main
import (
    _ "embed"
    "fmt"
)

//go:embed static
var embededFiles embed.FS

func main() {
	data, _ := embededFiles.ReadFile("static/version.txt")
	fmt.Println(string(data))
}
```



embed.FS 结构主要有3个对外方法：

```go
// Open 打开要读取的文件，并返回文件的 fs.File 结构.
func (f FS) Open(name string) (fs.File, error)

// ReadDir 读取并返回整个命名目录
func (f FS) ReadDir(name string) ([]fs.DirEntry, error)

// ReadFile 读取并返回 name 文件的内容.
func (f FS) ReadFile(name string) ([]byte, error)
```



注意：

- 文件夹分隔符采用正斜杠`/`,即使是 windows 系统也采用这个模式
- 使用的是相对路径，路径开头不可以带`/`；相对路径的根路径是 go 源文件所在的文件夹
- 支持使用双引号`"`或者反引号的方式应用到嵌入的文件名或者文件夹名或者模式名上，这对名称中带空格或者特殊字符的文件文件夹有用

```go
package main
import (
	"embed"
	"fmt"
)

//go:embed "he llo.txt" `hello-2.txt`
var f embed.FS

func main() {
	data, _ := f.ReadFile("he llo.txt")
	fmt.Println(string(data))
}
```



### 文件类型的匹配模式

`go:embed`指令中可以只写文件夹名，此文件夹中除了`.`和`_`开头的文件和文件夹都会被嵌入，并且子文件夹也会被递归的嵌入，形成一个此文件夹的文件系统。

| 通配符        | 释义                                                         |
| ------------- | ------------------------------------------------------------ |
| ?             | 代表任意一个字符（不包括半角中括号）                         |
| *             | 代表0至多个任意字符组成的字符串（不包括半角中括号）          |
| [...]和[!...] | 代表任意一个匹配方括号里字符的字符，!表示任意不匹配方括号中字符的字符 |
| [a-z]、[0-9]  | 代表匹配a-z任意一个字符的字符或是0-9中的任意一个数字         |
| **            | 部分系统支持，*不能跨目录匹配，**可以，不过目前个golang中和*是同义词 |

下面举一些例子，假设我们的项目在/tmp/proj：

```text
//go:embed images
这是匹配所有位于/tmp/proj/images及其子目录中的文件

//go:embed images/jpg/a.jpg
匹配/tmp/proj/images/jpg/a.jpg这一个文件

//go:embed a.txt
匹配/tmp/proj/a.txt

//go:embed images/jpg/*.jpg
匹配/tmp/proj/images/jpg下所有.jpg文件

//go:embed images/jpg/a?.jpg
匹配/tmp/proj/images/jpg下的a1.jpg a2.jpg ab.jpg等

//go:embed images/??g/*.*
匹配/tmp/proj/images下的jpg和png文件夹里的所有有后缀名的文件，例如png/123.png jpg/a.jpeg

//go:embed *
直接匹配整个/tmp/proj

//go:embed a.txt
//go:embed *.png *.jpg
//go:embed aa.jpg
可以指定多个//go:embed指令行，之间不能有空行，也可以用空格在一行里写上对个模式匹配，表示匹配所有这些文件，相当于并集操作
可以包含重复的文件或是模式串，golang对于相同的文件只会嵌入一次，很智能
```



如果想嵌入`.`和`_`开头的文件和文件夹， 比如 p 文件夹下的.hello.txt文件，那么就需要使用`*`，比如`go:embed p/*`。

`*`不具有递归性，所以子文件夹下的`.`和`_`不会被嵌入，除非你在专门使用子文件夹的`*`进行嵌入:

```go
package main

import (
	"embed"
	"fmt"
)

//go:embed p/*
var f embed.FS

func main() {
	data, _ := f.ReadFile("p/.hello.txt")
	fmt.Println(string(data))

	data, _ = f.ReadFile("p/q/.hi.txt") // 没有嵌入 p/q/.hi.txt
	fmt.Println(string(data))
}
```



嵌入和嵌入模式不支持绝对路径、不支持路径中包含`.`和`..`，如果想嵌入 go 源文件所在的路径，使用`*`:

```go
package main

import (
	"embed"
	"fmt"
)

//go:embed *
var f embed.FS

func main() {
	data, _ := f.ReadFile("hello.txt")
	fmt.Println(string(data))

	data, _ = f.ReadFile(".hello.txt")
	fmt.Println(string(data))
}
```



### 同一个文件嵌入为多个变量

比如下面的例子，s 和 S 变量都嵌入 hello.txt 的文件。

```go
package main
import (
	_ "embed"
	"fmt"
)

func main() {
	//go:embed hello.txt
	var s string

	//go:embed hello.txt
	var s2 string

	fmt.Println(s, s2)
}
```



局部变量 s 的值在编译时就已经嵌入了，而且虽然 s 和 s2 嵌入同一个文件，但是它们的值在编译的时候会使用初始化字段中的不同的值：

```shell
0x0021 00033 (/Users/....../main.go:10)        MOVQ    "".embed.1(SB), AX
0x0028 00040 (/Users/....../main.go:10)        MOVQ    "".embed.1+8(SB), CX
0x002f 00047 (/Users/....../main.go:13)        MOVQ    "".embed.2(SB), DX
0x0036 00054 (/Users/....../main.go:13)        MOVQ    DX, "".s2.ptr+72(SP)
0x003b 00059 (/Users/....../main.go:13)        MOVQ    "".embed.2+8(SB), BX

......

"".embed.1 SDATA size=16
       0x0000 00 00 00 00 00 00 00 00 0d 00 00 00 00 00 00 00  ................
       rel 0+8 t=1 go.string."hello, world!"+0
"".embed.2 SDATA size=16
       0x0000 00 00 00 00 00 00 00 00 0d 00 00 00 00 00 00 00  ................
       rel 0+8 t=1 go.string."hello, world!"+0
```

注意：此时虽然使用的是同一个文件，但在二进制包中，会包含两份`hello.txt`文档，所生成的二进制文件体积会比预期要大。



### exported/unexported的变量都支持

Go 可以将文件可以嵌入为 exported 的变量，也可以嵌入为 unexported 的变量。

```go
package main
import (
	_ "embed"
	"fmt"
)

//go:embed hello.txt
var s string

//go:embed hello2.txt
var S string

func main() {
	fmt.Println(s)

	fmt.Println(S)
}
```



### package级别的变量和局部变量都支持

前面的例子都是 package 一级的的变量，即使是函数内的局部变量，也都支持嵌入：

```go
package main
import (
	_ "embed"
	"fmt"
)

func main() {
	//go:embed hello.txt
	var s string

	//go:embed hello.txt
	var s2 string

	fmt.Println(s, s2)
}
```



### 文件类型时的只读

嵌入的内容是只读的。也就是在编译期嵌入文件的内容是什么，那么在运行时的内容也就是什么。

FS 文件系统值提供了打开和读取的方法，并没有 write 的方法，也就是说 FS 实例是线程安全的，多个 goroutine 可以并发使用。

```go
type FS
    func (f FS) Open(name string) (fs.File, error)
    func (f FS) ReadDir(name string) ([]fs.DirEntry, error)
    func (f FS) ReadFile(name string) ([]byte, error)
```



### 文件系统

`embed.FS`实现了 `io/fs.FS`接口，它可以打开一个文件，返回`fs.File`:

```go
package main
import (
	"embed"
	"fmt"
)

//go:embed *
var f embed.FS

func main() {
	helloFile, _ := f.Open("hello.txt")
	stat, _ := helloFile.Stat()
	fmt.Println(stat.Name(), stat.Size())
}
```

它还提供了 ReadFile 和 ReadDir 功能，遍历一个文件下的文件和文件夹信息：

```go
package main
import (
	"embed"
	"fmt"
)

//go:embed *
var f embed.FS

func main() {
	dirEntries, _ := f.ReadDir("p")
	for _, de := range dirEntries {
		fmt.Println(de.Name(), de.IsDir())
	}
}
```



因为它实现了`io/fs.FS`接口，所以可以返回它的子文件夹作为新的文件系统：

```go
package main
import (
	"embed"
	"fmt"
	"io/fs"
	"io/ioutil"
)

//go:embed *
var f embed.FS

func main() {
	ps, _ := fs.Sub(f, "p")
	hi, _ := ps.Open("q/hi.txt")
	data, _ := ioutil.ReadAll(hi)
	fmt.Println(string(data))
}
```



## embed的高级使用



### 简单静态Web服务

首先，在项目根目录下建立如下静态资源目录结构：

```shell
|- static
    |- js
    |   |- util.js
    |- img
    |   |- logo.jpg
    |- index.html
```



Web 服务器代码：

```go
package main

import (
    "embed"
    "io/fs"
    "log"
    "net/http"
    "os"
)


func main() {
    useOS := len(os.Args) > 1 && os.Args[1] == "live"
    http.Handle("/", http.FileServer(getFileSystem(useOS)))
    http.ListenAndServe(":8888", nil)
}

//go:embed static
var embededFiles embed.FS

func getFileSystem(useOS bool) http.FileSystem {
    if useOS {
        log.Print("using live mode")
        return http.FS(os.DirFS("static"))
    }

    log.Print("using embed mode")

    fsys, err := fs.Sub(embededFiles, "static")
    if err != nil {
        panic(err)
    }
    return http.FS(fsys)
}
```

以上代码，分别执行 go run . live 和 go run .，然后在浏览器中运行 http://localhost:8888，默认显示 static 目录下的index.html 文件内容。

当然，运行`go run . live` 和 `go run .` 的不同之处*在于**编译后的二进制程序文件在运行过程中是否依赖 static 目录中的静态文件资源。***



以下为验证步骤：

首先，使用编译到二进制文件的方式。



若文件内容改变，输出依然是改变前的内容，说明 embed 嵌入的文件内容在编译后不再依赖于原有静态文件了。

- 运行`go run .`
- 修改 index.html 文件内容为 `Hello China`
- 浏览器输入 http://localhost:8888 查看输出。输出内容为修改之前的`Hello World`

其次，使用普通的文件方式。若文件内容改变，输出的内容也改变，说明编译后依然依赖于原有静态文件。

- 运行`go run . live`
- 修改 index.html 文件内容为 `Hello China`
- 浏览器输入 http://localhost:8888 查看输出。输出修改后的内容：`Hello China`



### text/template和html/template.

同样的，template 也可以从嵌入的文件系统中解析模板：

```
func ParseFS(fsys fs.FS, patterns ...string) (*Template, error)
func (t *Template) ParseFS(fsys fs.FS, patterns ...string) (*Template, error)
```



### embed嵌入到Gin框架

要想将 Embed 投入当实际生产项目，那么我们就需要用到 embed.FS 这个数据类型，它表示存储多个文件和目录的结构。目前 gin 框架支持的静态文件的代码如下所示：

```go
func (group *RouterGroup) StaticFS(relativePath string, fs http.FileSystem) IRoutes
```


我们主流框架是还未支持 embed.FS 数据类型，所以需要做下改造，将 embed.FS 转换为 http.FileSystem：

```go
// 嵌入普通的静态资源
type webui struct {
   webuiEmbed embed.FS // 静态资源
   path       string   // 设置embed文件到静态资源的相对路径，也就是embed注释里的路径
}

// 静态资源被访问的核心逻辑
func (w *webui) Open(name string) (http.File, error) {
   if filepath.Separator != '/' && strings.ContainsRune(name, filepath.Separator) {
      return nil, errors.New("http: invalid character in file path")
   }
  
   fullName := filepath.Join(w.path, filepath.FromSlash(path.Clean("/"+name)))
   file, err := w.webuiEmbed.Open(fullName)
   wf := &WebuiFile{
      File: file,
   }
   return wf, err
}

type WebuiFile struct {
   io.Seeker
   fs.File
}

func (*WebuiFile) Readdir(count int) ([]fs.FileInfo, error) {
   return nil, nil
}
```



做完转换后，我们就可以使用gin返回embed提供的静态资源：

```go
// 设置简单的演示静态资源
webuiSimpleObj := &webui{
    webuiEmbed: simple.WebUI,
    path:       "webui",
}

// 设置简单的演示静态资源
handler.StaticFS("/webui/", webuiSimpleObj)
```

访问 http://127.0.0.1:8888/webui，可以看到数据。



### embed嵌入Ant Desgin

依葫芦画瓢，我们将整个 ant design 嵌入进来：

```go
// 设置ant design的路径，在config.ts里配置
handler.StaticFS("/ant/", webuiAntObj)

// 访问首页跳转到ant design的welcome页面
handler.GET("/", func(ctx *gin.Context) {
    ctx.Redirect(302, "/welcome")
    return
})

// Ant Design前端访问，try file到index.html
handler.GET("/welcome", func(context *gin.Context) {
    context.FileFromFS("/welcome", webuiAntIndexObj)
})
```

在设置前后端分离的项目，我们需要做 try file 到 index.html 操作，否则刷新页面，会走到后端，详细操作看 https://github.com/gotomicro/embedctl 



## embed的一些陷阱

方便的功能背后往往也会有陷阱相随，Golang 的内置静态资源嵌入也不例外。

### 隐藏文件的处理

根据2020年11月21日的[issue](https://github.com/golang/go/issues/42328)，现在 Golang 在对目录进行递归嵌入的时候会忽略名字以下划线（_）和点（.）开头的文件或目录。这些文件名在部分文件系统中为隐藏文件，issue的提出者认为默认不应该包含这些文件，隐藏文件通常包含对程序来说没有意义的元数据，或是用户的隐私配置，除非明确声明，否则嵌入资源中包含隐藏文件是不妥的。

举个例子，假设我们有个 images 文件夹，底下有`a.jpg`，`.b.jpg`两个常规文件，以及`_photo_metadata`和`pngs`两个子目录，根据最新的[commit](https://github.com/golang/go/commit/37588ffcb221c12c12882b591a16243ae2799fd1)，以下的嵌入资源指令的效果如注释中的解释：

```golang
//go:embed images
var images embed.FS // 不包含.b.jpg和_photo_metadata目录

//go:embed images/*
var images embed.FS // 注意！！！ 这里包含.b.jpg和_photo_metadata目录

//go:embed images/.b.jog
var bJPG []byte // 明确给出文件名也不会被忽略
```

注意第二条。使用`*`相当于明确给出了目录下所有文件的名字，因此点和下划线开头的文件和目录也会被包含。

当然，隐藏文件不止文件名特殊这么简单，在部分文件系统上拥有正常文件名的文件通过增加某些 flag 或者 attribute 也可以变为隐藏，目前怎么处理此类情况还没有定论。官方暂且按照社区的习惯使用文件名进行区分。

另外对于`*`是否应该包含隐藏文件的争论也没有停止，官方暂且认为应该包含隐藏文件，这点要多加注意。



### 资源是否应该被压缩

静态资源嵌入的提案被接受后争论最多的就是是否应该对资源采取压缩，压缩后的资源更紧凑，不会浪费太多存储空间，特别是一些大文本文件。同时更大的程序运行加载时间越长，cpu 缓存利用率可能会变低。

而反对意见认为压缩和运行时的解压一个浪费编译的时间一个浪费运行时的效率，在用户没有明确指定的情况下用户需要为自己不需要的功能花费代价。

目前官方采用的实现是不压缩嵌入资源，并预计在后续版本加入控制是否启用压缩的选项。



### 潜在的嵌入资源副本

前文中提到过重复的匹配和相同的文件 Golang 会自动只保留一份在变量中。没错，然而这是针对同一个变量的多个匹配说的，如果考虑下面的代码：

```golang
package main

import (
	_ "embed"
	"fmt"
)

//go:embed imgs/screenrecord.gif
var b []byte

//go:embed imgs/screenrecord.gif
var a []byte

func main() {
	fmt.Printf("a: %p %d\n", &a, len(a))
	fmt.Printf("b: %p %d\n", &b, len(b))
}
```

猜猜输出是什么：

```text
a: 0x9ff5a50 81100466
b: 0x9ff5a70 81100466
```

a 和 b 的地址不一样！那也没关系，我们知道 slice 是引用类型，底层说不定引用了同一个数组呢？那再来看看文件大小：

```bash
tree -sh .
.
├── [ 484]  embed_fs.go
├── [ 230]  embed_img2.go
├── [157M]  embed_img2
├── ...
├── [   0]  imgs
│   ├ ...
│   └── [ 77M]  screenrecord.gif
├── ...

4 directories, 19 files
```

程序是资源的两倍大，这差不多就可以说明问题了，资源被复制了一份。不过从代码的角度来考虑，a 和 b 是两个不同的对象，所以引用不同的数据也说的过去，但在开发的时候一定要小心，不要让两个资源集合出现交集，否则就要付出高昂的存储空间代价了。



### 过大的可执行文件带来的性能影响

程序文件过大会导致初次运行加载时间的增长，这是众所周知的。

然而过大的程序文件还可能会降低运行效率。程序需要利用现代的 cpu 快速缓存体系来提高性能，而更大的二进制文件意味着对于反复运行的热点功能 cpu 的快速缓存很可能会面临更多的缓存失效，因为缓存的大小有限，需要两次三次的读取和刷新才能运行完一个热点代码片段。这就是为什么几乎所有的编译器都会自行指定函数是否会被内联化而不是把这种控制权利移交给用户的原因。

然而嵌入静态文件之后究竟会对性能有多少影响呢？目前缺乏实验证据，所以没有定论。

通过修改二进制文件的一部分格式也可以让代码部分和资源部分分离从而代码在 cpu 看来更加紧凑，当然这么做会不会严重破坏兼容，是否真的有用也未可知。



### 会被忽略的目录

前面说过，embed 会递归处理目录，除了以下的几个：

- .git
- .svn
- .bzr
- .hg

这些都是版本控制工具的目录，资源里理应不包含他们，因此是被忽略的。会被忽略的目录列在`src/cmd/go/internal/load/pkg.go`的`isBadEmbedName`函数里。

**.idea不在此列，小心！！！！**

另外不像隐藏文件可以明确指定嵌入，这些目录你是无法用任何正常手段嵌入的，Golang 都会忽略他们。



## 总结

使用 Golang1.16 你可以更轻松地创建嵌入资源，不过在享受便利的同时也要注意利弊取舍，使用 docker 管理资源和部署也不失为一种好方法。



## 参考

- [How to Use //go:embed](https://blog.carlmjohnson.net/post/2021/how-to-use-go-embed/)
- [golang1.16内嵌静态资源指南](https://www.cnblogs.com/apocelipes/p/13907858.html)
- [Go embed 简明教程](https://colobu.com/2021/01/17/go-embed-tutorial/)




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))
