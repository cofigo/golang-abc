# 说明

虽然 Go 语言标准库里面有 [flag](https://strconv.com/posts/flag/) 包可以作为命令行解析工具，但是 flag 包的用法有限，举例几个场景：

1. 子命令：flag 虽然也可以支持子命令，如`hugo server`，但是写起来非常麻烦。对于三级甚至更多级别的子命令更不支持
2. 仅支持`-` 开头参数：对于两个`--`开头的方式无法支持，例如`hugo --debug`
3. 无法同时支持`-`与`--`开头的参数：例如 hugo，支持`-h`和`--help`，都是显示帮助信息

所以在实际项目中可以引入第三方命令行工具更好的支持这些场景。目前 Go 语言中流行的工具库包含`spf13/cobra`和`urfave/cli`。



## urfave/cli

[urfave/cli](https://github.com/urfave/cli) 是一个简单、快速的命令行程序开发框架，使用它的知名项目包含 Gogs、Drone、Gitea 等。



### 安装库

```bash
$ go get github.com/urfave/cli
```



### 使用

#### 创建单一命令行工具

```go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/urfave/cli"
)

var (
	flags []cli.Flag
	host  string
	port  int
)

func init() {
	flags = []cli.Flag{
		cli.StringFlag{
			Name:        "t, host, ip-address",
			Value:       "127.0.0.1",
			Usage:       "Server host",
			Destination: &host,
		},
		cli.IntFlag{
			Name:        "p, port",
			Value:       8000,
			Usage:       "Server port",
			Destination: &port,
		},
	}
}

func main() {
	app := cli.NewApp()
	app.Name = "cliDemo"
	app.Usage = "Application Usage"
	app.HideVersion = true
	app.Flags = flags
	app.Action = Action

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}

func Action(c *cli.Context) error {
	if c.Int("port") < 1024 {
		cli.ShowAppHelp(c)
		return cli.NewExitError("Ports below 1024 is not available", 2)
	}

	fmt.Printf("Listening at: http://%s:%d", host, c.Int("port"))
	return nil
}
```

flags 是`cli.Flag`结构列表，包含主机名和端口2项 Flag，每项都指定了 Name (可以有多个，用逗号隔开)、Value (默认值)、Usage (帮助信息)、Destination (对应变量的内存地址，因为在命令行内会修改变量)。要注意 Flag 需要使用对应类型 xxxFlag（如cli.StringFlag、cli.IntFlag)

在 main 函数中首先用`cli.NewApp()`创建一个新的 app，Name、Usage等项的值在帮助输出中都有体现，HideVersion 是用于隐藏版本号，而 Action 表示「行为」，相当于 Parse 参数后的最终输出，在这里我把 Action 独立成了一个函数，里面的逻辑：

1. 如果端口号的值小于1024会先当因帮助信息，再抛错
2. 打印主机和端口的值，这里用了2种方法：hots 直接用的是变量；而 port 用了`c.Int("port")`的方式获取，要注意对应参数项的类型，如果获得 host 的值需要用`c.String("host")`

程序执行输出如下：

```bash
$ go run main.go
Listening at: http://127.0.0.1:8000

$ go run main.go -h
NAME:
   cliDemo - Application Usage

USAGE:
   use_flags [global options] command [command options] [arguments...]

COMMANDS:
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   -t value, --host value, --ip-address value  Server host (default: "127.0.0.1")
   -p value, --port value                      Server port (default: 8000)
   --help, -h                                  show help

$ go run mian.go -t 172.16.0.1 --port 8080
Listening at: http://172.16.0.1:8080

$ go run mian.go -t 172.16.0.1 --port 21
NAME:
   cliDemo - Application Usage

USAGE:
   use_flags [global options] command [command options] [arguments...]

COMMANDS:
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   -t value, --host value, --ip-address value  Server host (default: "127.0.0.1")
   -p value, --port value                      Server port (default: 8000)
   --help, -h                                  show help
Ports below 1024 is not available
exit status 2
```

可以看到不同参数下执行效果和输出是不一样的。



#### 创建含子命令的命令行工具

命令行参数另外一个重要场景是子命令(Subcommand)，基于官网文档看一下例子：

```go
package main

import (
    "fmt"
    "log"
    "os"

    "github.com/urfave/cli"
)

func main() {
    app := cli.NewApp()

    app.Commands = []cli.Command{
        {
            Name:     "add",
            Aliases:  []string{"a"},
            Usage:    "add a task to the list",
            Category: "Add",
            Action: func(c *cli.Context) error {
                fmt.Println("added task: ", c.Args().First())
                return nil
            },
        },
        {
            Name:     "template",
            Aliases:  []string{"t", "tmpl"},
            Usage:    "options for task templates",
            Category: "Template",
            Subcommands: []cli.Command{
                {
                    Name:  "add",
                    Usage: "add a new template",
                    Action: func(c *cli.Context) error {
                        fmt.Println("new task template: ", c.Args().First())
                        return nil
                    },
                },
                {
                    Name:  "remove",
                    Usage: "remove an existing template",
                    Action: func(c *cli.Context) error {
                        fmt.Println("removed task template: ", c.Args().First())
                        return nil
                    },
                },
            },
        },
    }

    app.Name = "cliDemoSub"
    app.Usage = "application usage"
    app.Description = "application description"  // 描述
    app.Version = "1.0.1" // 版本

    err := app.Run(os.Args)
    if err != nil {
        log.Fatal(err)
    }
}
```

`urfave/cli`支持分组子命令，这个例子中包含2个子命令，分别是 add (在 Add 组)和 template (在 Template 组)，其中 template 子命令下也有2个子命令 add / remove。子命令支持别名，需要放在 Aliases 里面，分组需要放在 Category 里面。

另外这次设置了描述和版本号，会在帮助信息中展示出来，而前一个例子隐藏版本部分信息了。

子命令的行为由 Action 决定，它的值是一个函数，可以看代码了解。

程序执行输出如下：

```bash
$ go run main.go
NAME:
   cliDemoSub - application usage

USAGE:
   use_subcommand [global options] command [command options] [arguments...]

VERSION:
   1.0.1

DESCRIPTION:
   application description

COMMANDS:
     help, h  Shows a list of commands or help for one command

   Add:
     add, a  add a task to the list

   Template:
     template, t, tmpl  options for task templates

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version

$ go run main.go add abc
added task:  abc

$ go run main.go template --help
NAME:
   cliDemoSub template - options for task templates

USAGE:
   cliDemoSub template command [command options] [arguments...]

COMMANDS:
     add     add a new template
     remove  remove an existing template

OPTIONS:
   --help, -h  show help

$ go run main.go template add bcd
new task template:  bcd

$ go run main.go template remove cde
removed task template:  cde

$ go run main.go --version
cliDemoSub version 1.0.1
```



### 代码地址

完整代码可以在[这个地址](https://github.com/cofigo/go-examples/tree/main/70.开源框架/cli)找到。



## 参考

- [urfave/cli](https://github.com/urfave/cli)
- [Golang第三方命令行工具](https://strconv.com/posts/cli/)



----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/cofigo/golang-abc.git))

