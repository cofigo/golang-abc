# 解决 go get golang.org/x 包安装失败问题



## 问题描述

因为golang.org相关网站在国内无法正常访问，会导致我们在使用`go get`、`go mod`等命令下载软件包时，下载失败，类似如下：

```
$ go get -u golang.org/x/sys

go get golang.org/x/sys: unrecognized import path "golang.org/x/sys" (https fetch: Get https://golang.org/x/sys?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)
```



## 解决方案

### 方案一：使用代理服务

从 `Go 1.11` 版本开始，官方支持了 `go module` 包依赖管理工具。并新增了 `GOPROXY` 环境变量。如果设置了该变量，下载源代码时将会通过这个环境变量设置的代理地址，而不再是直接从代码库下载。

可以使用的GOPROXY代理服务地址：

- [goproxy.io](http://goproxy.io) : 开源项目提供的代理服务
- [goproxy.cn](http://goproxy.cn) : 由七牛提供的国内代理服务，方便国内用户



如果需要启用，只需要设置环境变量`GOPROXY`即可以，类似如下：

```
export GOPROXY=https://goproxy.io
```

也可以配置两个服务器：

```
GOPROXY=https://goproxy.cn,https://goproxy.io,direct
```



不过，**需要依赖于 `go module` 功能**。可通过设置 `GO111MODULE` 环境变量开启 MODULE：

```
export GO111MODULE=on
```



如果项目不在 `GOPATH` 中，则无法使用 `go get ...`，但可以使用 `go mod ...` 相关命令。

也可以通过置空这个环境变量来关闭，`export GOPROXY=`。

我们推荐使用 `GOPROXY` 设置代理服务的方式，但前提是 `Go`版本 **>= 1.11**。



### 方案二：手动下载软件包

我们常见的 `golang.org/x/...` 包，一般在 GitHub 上都有官方的镜像仓库对应。比如 `golang.org/x/text` 对应 `github.com/golang/text`。所以，我们可以手动下载或 clone 对应的 GitHub 仓库到指定的目录下。

如下载`sys`软件包：

```
mkdir $GOPATH/src/golang.org/x 
cd $GOPATH/src/golang.org/x 

git clone git@github.com:golang/sys.git 
rm -rf sys/.git
```

当如果需要指定版本的时候，该方法就无解了，因为 GitHub 上的镜像仓库多数都没有 tag。



### 方案三：科学上网

该方式仅针对有个人代理服务器的开发者。可以在个人脚本配置文件`.zshrc`中增加：

```
alias proxyon="export http_proxy=http://127.0.0.1:1087; export https_proxy=http://127.0.0.1:1087"
alias proxyoff="unset http_proxy; unset https_proxy"
```

其中，`127.0.0.1:1087`及`127.0.0.1:1087`为个人代理地址及端口。



则，在需要启用代理服务器时，直接在终端中输入`proxyon`即可；使用完毕后，可以使用命令`proxyoff`关闭。



### 其他：使用 go mod replace

从 `Go 1.11` 版本开始，新增支持了 `go modules` 用于解决包依赖管理问题。该工具提供了 `replace`，就是为了解决包的别名问题，也能替我们解决 `golang.org/x` 无法下载的的问题。

`go module` 被集成到原生的 `go mod` 命令中，但是如果你的代码库在 `$GOPATH` 中，`module` 功能是默认不会开启的，想要开启也非常简单，通过一个环境变量即可开启 `export GO111MODULE=on`。

以下为`go.mod`文件参考示例：

```
module example.com/hello 

require (
	golang.org/x/text v0.3.0 
) 

replace (
	golang.org/x/text => github.com/golang/text v0.3.0 
)
```

> `go.mod`文件应当位于项目根目录。



----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))