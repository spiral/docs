# 控制台命令

Spiral 框架一些命令来协助开发和控制构建。

列出所有可用的控制台命令：

```bash
$ php app.php
```

查看特定命令的帮助：

```bash
$ php app.php help db:table
```

也可以使用简化的格式：

```bash
$ php app.php db:table -h
```

> 如果需要创建自己的命令，可以查看[相关章节](/zh_CN/console/commands.md)。

## 别名

Spiral 控制台的底层是 `Symfony/Console`, 因此开发者可以使用命令名称缩写，只要提供的缩写足以定位到目标命令即可：

```bash
# 解析到 `update` 命令
$ php app.php up 

# `configure`, `cache:clean`, `cycle:*` 命令都匹配，造成了冲突。
$ php app.php c
```

## 配置

Spiral 应用包含了一个核心命令 `configure`. 这个命令会按顺序执行一系列操作，以确保应用已正确配置，创建了必备目录并验证资源的权限正确。

可以在执行命令的时候输出更详细的信息：

```bash
$ php app.php configure -vv
```

> 新创建（安装）的应用，一定要在首次运行前执行一次 `configure` 命令。

## 应用服务器

应用服务器有它自己的一系列命令，要列出所有可用的服务器命令，可以执行：

```bash
$ ./spiral
```

要了解有关应用服务器命令的更多信息，可以查看[相关文档](https://roadrunner.dev/docs/beep-beep-cli)。
