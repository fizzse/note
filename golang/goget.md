## go get 私有仓库

### go mod
Go1.10后退出go mod包管理。依赖的非标准库都在go.mod文件中进行记录,内容如下图所示。

```go
module github.com/fizzse/gobase

go 1.16

require (
	github.com/uber/jaeger-lib v2.4.1+incompatible // indirect
	github.com/ugorji/go v1.2.5 // indirect
	go.uber.org/zap v1.16.0
)
```

当我们需要使用某个第三方库时，就是通过go get go.uber.org/zap 下载依赖包，并自动在go.mod文件中记录。这里引申出几个问题

1. go get go.uber.org/zap 的工作原理是什么？  
go get 其实就是 git clone + go install。clone默认使用https的方式，如：go get go.uber.org/zap 其实本质为 git clone https://go.uber.org/zap.git

2. 我们自己写的代码包 module应该怎么命名？  
如果代码包不会被别的包依赖以及从网络拉取。怎么命名都可以。但是如果想作为工具包被别的代码块导入，那么命名必须规范。 
了解了go get的本质后，module应该怎么命名就很清楚了。只要你定义的包名可以正确的被转化成可别下载的git仓库即可。如我有个仓库名为gobase在github上
进行保持，且仓库在fizzse这个用户下。那么名字就应该为:
```shell
github.com/fizzse/gobasse
```   
如果在gitee.com上面，那么应该定义为gitee.com/fizzse/gobasse才可以被go get 下来
   
3. 怎么go get 私有仓库的代码？  
通过上面两个问题的阐述，如何go get 私有仓库思路应该很清晰了。就是如果你可以git clone xxx。那么你就可以go get xxx。由于默认是https的方式需要输入密码，
所以我们只要将https替换为ssh方式(前提配置ssh私钥)即可。

```shell
git config --global url."git@github.com:".insteadOf "https://github.com/"
export GOPRIVATE=git@github.com:fizzse
```   

如果是自建git仓库同理,拿gitee举例：
```shell
git config --global url."git@gitee.com:".insteadOf "https://gitee.com/"
export GOPRIVATE=git@gitee.com:fizzse
```