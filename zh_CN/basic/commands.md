# 控制台命令
Spiral 框架提供了大量的命令来应用服务器以及协助开发。

查看所有可用的命令列表：

```bash
$ php app.php
```

查看某一个命令的使用帮助：

```bash
$ php app.php help db:table
```

也可以用另一种方式：

```bash
$ php app.php db:table -h
```

> 在后续文档中，有介绍如何创建自己的命令。

## 命令别名
Spiral 控制台组件是基于 `Symfony/Console` 构建的，因此你可以使用命令的缩写/别名，只要提供的缩写足以找出目标的命令即可：

```bash
# 这个缩写会被解析为 `update`
$ php app.php up 

# 匹配到的多个命令 `configure`, `cache:clean`, `cycle:*` 产生了冲突。
$ php app.php c
```

## 配置命令
Spiral 提供的应用程序模板包含了一个主要命令 `configure`. 这个命令会执行一系列操作以确保应用已经正确安装依赖，创建了必要的目录并验证目录权限。

执行下面的命令可以查看 `configure` 命令的详细输出：

```bash
$ php app.php configure -vv
```

> 在运行一个新安装/创建的应用之前，一定要先执行 `configure` 命令。

## 应用服务器命令
应用服务器有它自己的命令集，要列出所有可用的服务器命令，可以执行：

```bash
$ ./spiral
```

关于服务器命令的更多信息，可以查阅 [roadrunner 文档](https://roadrunner.dev/docs/usage-server-commands)。
