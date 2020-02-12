
# 框架 - 应用服务器

Spiral 框架的应用服务器基于 [RoadRunner](https://roadrunner.dev), 但包含了特定于 Spiral 的多个附加特性，例如队列、调度器以及 GRPC 集成。

> 注意：如果要定制应用服务器，你需要具备一些 Golang 的基础知识。

## 下载应用服务器

你可以从项目的[发布页面](https://github.com/spiral/framework/releases)直接下载应用服务器。

如果你的 PHP 安装包含 `php-cli` 和 `php-zip` 扩展，那么也可以让 spiral 自动下载应用服务器：

```bash
$ ./vendor/bin/spiral get
```

## 运行服务器

Spiral 服务器可以通过以下命令快速运行起来，默认监听端口是 `:8080`:

```bash
$ ./spiral serve -v -d
```

你还可以通过下面的命令，监控工作进程的实时内存占用以及其它信息：

```bash
$ ./spiral http:workers -i
```

> 除了 http 工作进程外，也可以用类似的 `jobs:workers` 和 `grpc:workers` 命令检查其它的调度器。

## 构建应用服务器

本文档的大部分章节将用于解释如何通过添加你自己的 RoadRunner 服务、中间件和数据提供程序来扩展应用程序功能。学习如何自行构建应用服务器也是很必要的。

> 如果只是使用 Spiral 框架，并不需要你去学习 Golang 或者自行构建应用服务器。默认的构建版本已经涵盖了框架所需的所有特性。

#### 安装 Golang

要构建应用服务器，你的机器上必须安装有 [Golang 1.12 以上版本](https://golang.org/dl/)。

### 创建 main.go 文件

下载默认的 [main.go](https://github.com/spiral/framework/blob/master/main.go) 文件，你可以编辑这个文件来注册自己的服务。你可以把该文件放在你的项目根目录或者其它任何地方。

### 安装 go 的依赖模块

Go Modules 是 Golang 使用的依赖管理器，和 Composer 非常相似。首先要初始化一个空的 `go.mod` 文件（相当于 `composer.json`），可以在 `main.go` 所在的目录下执行下面的命令：

```bash
$ go mod init {代码仓库名}
``` 

如果你计划安装自定义扩展，请确保 `{代码仓库名}` 指向一个 Golang 可以下载的仓库，例如 `github.com/username/my-application`.

### 构建服务器

一切准备就绪之后，执行以下命令即可构建自定义版本的应用服务器：

```bash
$ go build
```
