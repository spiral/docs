# 安装

你可以单独安装 Spiral 的各个组件来构建自己的项目，也可以根据需要通过预创建的项目框架来创建项目，这些项目框架已经整合了必要的依赖项。比如你可以下载 Web 项目框架或者命令行项目框架，基于它们启动一个新的应用的开发。

默认的 Web 全栈项目框架可以在 https://github.com/spiral/app 找到。它会安装一个 RoadRunner 应用服务器的定制版本，以及其他一些扩展，并启用了框架相关的大量功能。

<br/>

环境要求
--------
请确认你的服务器至少安装了下面版本的 PHP 和扩展：
* PHP 7.2+, 64bit
* *mb-string* (spiral 是以 UTF-8 为核心编码的框架)
* PDO 扩展及要使用的数据库驱动

Web 应用包
--------
应用包集成了以下组件：
* 基于 [RoadRunner](https://roadrunner.dev) 的高性能 HTTP, HTTP/2 服务器 
* 通过 Symfony/Console 提供的控制台命令
* AMQP, Beanstalk, Amazon SQS, in-Memory 队列支持
* Stempler 模板引擎
* 通过 Symfony/Translation 提供的多语言支持
* 安全, 验证, 过滤模型
* PSR-7 风格的 HTTP 流水线, session, 加密的 cookies
* DBAL 和 数据迁移支持
* Monolog, Dotenv
* Prometheus metrics
* [Cycle DataMapper ORM](https://github.com/cycle)

创建项目
--------
```bash
composer create-project spiral/app myProject
```

>  应用服务器会自动下载（需要 `php-curl` 和 `php-zip`）。

一旦项目创建和依赖安装完成，你可以进入项目根目录执行以下的命令来确保项目被正确配置：

```bash
$ php ./app.php configure
```

要启动应用服务器，可以执行（ Mac 和 Linux ）：

```bash
$ ./spiral serve -v -d
```

在 Windows 系统，用下面的命令代替：

```bash
$ spiral.exe serve -v -d
```

现在你的应用已经可以通过 `http://localhost:8080` 访问了。

> 你可以在[这里](https://roadrunner.dev/docs)找到有关应用服务器配置的更多信息。

## 其它项目框架
除了 Web 项目框架以外，还有其它应用项目框架可供使用：
- https://github.com/spiral/app-cli - 极简命令行应用的项目框架
- https://github.com/spiral/app-grpc - GRPC 应用项目框架（不含 Views 和 HTTP ）
