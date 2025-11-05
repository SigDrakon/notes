









# 1. Go Modules 简介

在引入 Go Modules 之前，Go 语言的包管理依赖 GOPATH，所有的代码和依赖库都必须放在 GOPATH 目录下，这导致项目之间的包依赖容易混淆，版本控制困难。

Go Modules 的出现，使得 Go 项目不再依赖 GOPATH，项目目录可以放在任意位置，并且每个项目的依赖包都有自己的版本控制信息，解决了包冲突的问题。

Go Modules 主要有以下优点：

- 支持在项目中独立管理依赖包，不受 GOPATH 限制。
- 自动管理依赖包的版本。
- 支持版本回滚、升级和降级等操作。
- 提供更加灵活的包管理方式，方便多人协作开发。



# 2. Go Modules 的基本使用





## 2.1. 项目初始化

要使用 Go Modules，首先需要初始化一个 Go Modules 项目。执行以下命令即可：

```bash
go mod init <module-name>
```





