# 控制台 - 安装和配置

默认情况下，所有官方提供的项目模板都已经包含了控制台组件。但如果是在其它项目（比如自己组装的项目）中使用这个组件，别忘了用 composer 安装 `spiral/console` 依赖，并且修改应用引导程序：

```php
[
    //...
    Spiral\Bootloader\CommandBootloader::class,
]
```

务必在最后（在包含命令定义的引导程序之后）启用这个引导程序，因为它在引导时会激活之前添加的组件中包含的所有命令。

要调用应用命令，执行下面的命令：

```bash
$ php app.php command:name
```

查看所有可用的命令列表，执行下面的命令：

```bash
$ php app.php
```

查看特定命令的使用帮助：

```bash
$ php app.php help command:name
```

## 在应用中调用命令

在应用程序代码或者测试代码中调用控制台命令。这在创建用于测试的模拟数据或者自动预配置数据库时非常有用。

要在代码中调用控制台命令，可以用过 `Spiral\Console\Console` 来实现：

```php
use Spiral\Console\Console;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

// ...

public function test(Console $console)
{
    $input = new ArrayInput(['args' => 'value']);
    $output = new BufferedOutput();
    
    return $console->run($command, $input, $output);
}
```

## Symfony/Console

Spiral 的控制台调度器是在强大的 [Symfony Console](https://symfony.com/doc/current/components/console.html) 组件基础上构建的。

> 开发者在命令行应用中可以注册原生的 Symfony 命令。

## 配置

要对控制台组件进行配置，可以使用 `Spiral\Config\ConfiguratorInterface` 接口或者在 `app/config/console.php` 创建一个配置文件。

```php
return [
     // 应用名称
     'name'      => null,
     
     // 应用版本号
     'version'   => null,
     
     // 应用的命令列表（如果禁用了自动发现）
     'commands'  => [],
     
     // 执行 `app configure` 时要执行的命令和顺序
     'configure' => [],
     
     // 执行 `app update` 时要执行的命令和顺序
     'update'    => []
];
```

在程序引导阶段可以通过 `Spiral\Bootloader\ConsoleBootloader` 引导程序修改上面的部分配置值。比如注册新的用户命令：

```php
public function boot(ConsoleBootloader $console)
{
    $console->addCommand(MyCommand::class);
}
```

> 提示：默认情况下控制台组件使用自动发现模式来自动发现 `app/` 目录下的所有用户命令。

如果要在配置（configure）或者升级（update）要执行的命令序列中增加命令：

```php
public function boot(ConsoleBootloader $console)
{
  $console->addUpdateSequence('my:command', '<info>Running my:command...</info>');
}
```

## 与 RoadRunner 关联

特别提醒，控制台命令是在 RoadRunner 服务器之外调用。如果某个命令要与应用服务器通讯，必须先运行一个应用服务器实例。
