# 说明

虽然 Go 语言标准库里面有 [flag](https://strconv.com/posts/flag/) 包可以作为命令行解析工具，但是 flag 包的用法有限，举例几个场景：

1. 子命令：flag 虽然也可以支持子命令，如`hugo server`，但是写起来非常麻烦。对于三级甚至更多级别的子命令更不支持
2. 仅支持`-` 开头参数：对于两个`--`开头的方式无法支持，例如`hugo --debug`
3. 无法同时支持`-`与`--`开头的参数：例如 hugo，支持`-h`和`--help`，都是显示帮助信息

所以在实际项目中可以引入第三方命令行工具更好的支持这些场景。目前 Go 语言中流行的工具库包含`spf13/cobra`和`urfave/cli`。



## spf13/cobra

[spf13/cobra](https://github.com/spf13/cobra) 既是一个用来创建强大的现代 CLI 命令行的 Golang 库，也是一个生成程序应用和命令行文件的工具。Docker、Kubernetes、Hugo、 Etcd 等程序都使用了它构建命令。



### 安装库

```bash
$ go get -u github.com/spf13/cobra/cobra
```



### 使用

使用 cobra 库，可以使用 cobra 命令直接初始化工程，也可以在已有的代码工程中进行集成。



#### 创建单一命令行工具

可以直接使用 cobra 命令生成一个空的命令行工具工程：

```bash
$ mkdir ./cliDemo
$ cd ./cliDemo
$ cobra init --pkg-name cliDemo  # 创建一个叫做 cliDemo 的应用
$ tree .  # 在当前目录下会创建一个cmd(里面只包含root.go)和一个main.go
.
├── LICENSE
├── cmd
│   └── root.go
└── main.go

1 directory, 3 file
```

初始化命令行工具的模块依赖：

```bash
$ go mod init cliDemo # 初始化为本地工程
$ go mod tidy # 根据需要解析并下载依赖库
$ go mod vendor # 将依赖库本地化拷贝至工程目录的下的 vendor 子目录（非必须）
```

直接使用 `run` 命令运行：

```shell
$ go run main.go
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
```

这样可以看到默认的输出了。

相关代码类似如下：

```go
// main.go
package main

import "cobraDemo/cmd"

func main() {
  cmd.Execute()
}

// cmd/root.go
package cmd

import (
  "fmt"
  "os"
  "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
  Use:   "cobraDemo",
  Short: "A brief description of your application",
  Long: `A longer description...`
}

func init() {
  cobra.OnInitialize(initConfig)  // viper是cobra集成的配置文件读取的库，以后我们会专门说

  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobraDemo.yaml)") // 添加`--config`参数。它是全局的~


  // Cobra also supports local flags, which will only run
  // when this action is called directly.
  rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle") // 添加一个布尔值的参数toggle, 在命令行可以用`-t`或者`--toggle`指定
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
```

其中：

1. newCmd.PersistentFlags。全局参数，根命令和子命令下都包含了这个参数项
2. newCmd.Flags。本地参数，只对当前命令下生效，其他子命令下不会继承



#### 创建含子命令的命令行工具

可以直接使用 cobra 命令生成一个空的命令行工具工程：

```bash
$ mkdir ./cliDemoSub
$ cd ./cliDemoSub
$ cobra init --pkg-name cliDemoSub  # 创建一个叫做 cliDemoSub 的应用
$ cobra add new # 为命令行工具增加一个子命令`new`

$ tree . 
.
├── LICENSE
├── cmd
│   ├── new.go
│   └── root.go
└── main.go

1 directory, 4 file
```

可以看到比单一命令行工具多了`cmd/new.go`文件，该文件即为子命令代码文件。

初始化命令行工具的模块依赖：

```bash
$ go mod init cliDemoSub # 初始化为本地工程
$ go mod tidy # 根据需要解析并下载依赖库
$ go mod vendor # 将依赖库本地化拷贝至工程目录的下的 vendor 子目录（非必须）
```

直接使用 `run` 命令运行：

```shell
$ go run main.go
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cliDemoSub [command]

Available Commands:
  completion  generate the autocompletion script for the specified shell
  help        Help about any command
  new         A brief description of your command

Flags:
      --config string   config file (default is $HOME/.cliDemoSub.yaml)
  -h, --help            help for cliDemoSub
  -t, --toggle          Help message for toggle

Use "cliDemoSub [command] --help" for more information about a command.
```

这样可以看到默认的输出了。

子命令帮助信息：

```shell
$ go run main.go new --help
A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cliDemoSub new [flags]

Flags:
  -h, --help   help for new

Global Flags:
      --config string   config file (default is $HOME/.cliDemoSub.yaml)
```

`cmd/new.go`文件，去掉注释和空行如下：

```go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
)
var newCmd = &cobra.Command{
    Use:   "new",
    Short: "A brief description of your command",
    Long: `A longer description...`,
    Run: func(cmd *cobra.Command, args []string) {
           fmt.Println("new called")
    },
}
func init() {
    rootCmd.AddCommand(newCmd)
}
```

添加命令就是在 init 函数中执行`rootCmd.AddCommand`，而 newCmd 中比之前看的 rootCmd 多了一个 Run 成员，指定执行子命令时要做什么，这样整个命令应用就可以用了。



#### 代码集成

我们也可以直接在程序里面集成 cobra，就像上面做的代码解析，无非就是用`cobra.Command`创建一个 rootCmd，用`rootCmd.AddCommand`添加子命令，在对应代码文件的init函数中定义 flag。

在 new 子命令下体验定义 flag 的方法(都用`newCmd.Flags`)：

```go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
)

var (
    n, a int
    s string
)

var newCmd = &cobra.Command{
    Use:   "new",
    Short: "A brief description of your command",
    Long: `A longer description...`,
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Printf("n is %v\n", n)
        fmt.Printf("a is %v\n", a)
        fmt.Printf("s is %v\n", s)
        q, _ := cmd.Flags().GetBool("q")
        fmt.Printf("q is %v\n", q)
        bbb, _ := cmd.Flags().GetInt("bbb")
        fmt.Printf("bbb is %v\n", bbb)
    },
}

func init() {
    rootCmd.AddCommand(newCmd)

    newCmd.Flags().IntVar(&n, "intf", 0, "Set Int")
    newCmd.Flags().StringVar(&s, "stringf", "sss", "Set String")
    newCmd.Flags().Bool("q", false, "Set Bool")

    newCmd.Flags().IntVarP(&a, "aaa", "a", 1, "Set A")
    newCmd.Flags().IntP("bbb", "b", -1, "Set B")
}
```

cobra 增加 flag 的函数主要有以下几种形式：

- xxxVar()：第一个参数指定变量地址，参数解析后直接存储在该变量中
- xxx()：不指定变量地址，参数使用`cmd.Flags().Getxxx("yyy")`方式解析并作为返回值
- xxxP() / xxxVarP()：类似以上两种，但增加指定段参数，即该方式增加的定义同时支持**短参数**和**长参数**



修改后命令行工具的帮助信息：

```bash
$ go run main.go new -h
A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cliDemoSub new [flags]

Flags:
  -a, --aaa int          Set A (default 1)
  -b, --bbb int          Set B (default -1)
  -h, --help             help for new
      --intf int         Set Int
      --q                Set Bool
      --stringf string   Set String (default "sss")

Global Flags:
      --config string   config file (default is $HOME/.cobraDemo.yaml)
```

带参数使用命令行工具：

```shell
$ go run main.go new
n is 0
a is 1
s is sss
q is false
bbb is -1
$ go run main.go new -a 1 --bbb=2 --intf 3 --stringf="abc" --q
n is 3
a is 1
s is abc
q is true
bbb is 2
```



####  使用说明

有关默认配置项：

1. `--config`是全局的，根命令和子命令下都包含了这个参数项
2. `-h`(`--help`)是默认参数项
3. 根命令和子命令可以配置需要的参数项



## 参考

- [spf13/cobra](https://github.com/spf13/cobra)
- [Golang第三方命令行工具](https://strconv.com/posts/cli/)



----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/cofigo/golang-abc.git))

