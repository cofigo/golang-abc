# Windows下Go程序添加图标

计划使用 Go 语言编译一系列实用工具，提高自己的工作效率。发现编译后的`.exe`文件没有图标，甚是难看，所以找了 windows 平台下添加 Go 程序图标的方法。



## 查找ico图标

 查找一个符合程序气质的图标，下载备用。

 ico链图标下载： [easyicon](https://www.easyicon.net/)



## 生成syso文件

 rsrc 是在 Windows 的 Go 程序中嵌入`.ico`和`manifest`资源的工具。



### 下载安装rsrc

```shell
$ go get github.com/akavel/rsrc
```



### 生成程序描述文件ico.manifest

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
<assemblyIdentity
    version="1.0.0.0"
    processorArchitecture="x86"
    name="controls"
    type="win32"
></assemblyIdentity>
<dependency>
    <dependentAssembly>
        <assemblyIdentity
            type="win32"
            name="Microsoft.Windows.Common-Controls"
            version="6.0.0.0"
            processorArchitecture="*"
            publicKeyToken="6595b64144ccf1df"
            language="*"
        ></assemblyIdentity>
    </dependentAssembly>
</dependency>
</assembly>
```



### 生成go程序嵌入文件

```shell
$ rsrc.exe -manifest ico.manifest -o myapp.syso -ico myapp.ico
```



### 提示

- go get 下载的 rsrc.exe 记得添加到环境变量 PATH 中，以免无法执行 rsrc.exe。
- go get 如果下载较慢，可以通过 go mod 方式下载。
- 生成的 myapp.syso 记得放到源码文件目录中。 



## 编译Go程序

 将 myapp.syso 文件放到相应 Go 程序下，然后直接运行`go build .`即可。

 Golang 已经可以自动寻找子目录下的 syso 文件。



## 参考

- [Windows下Go程序添加图标](https://blog.51cto.com/9406836/2490936)




----

本文原始来源 [Endial Fang](https://github.com/endial) @ [Github.com](https://github.com) ([项目地址](https://github.com/endial/study-golang.git))
