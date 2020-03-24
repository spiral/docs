# 控制台 - 用户命令

开发者可以向自己的应用添加新的控制台命令，或者通过引导程序注册插件提供的命令。默认配置下，控制台组件会自动发现 `app/src` 目录下的命令。

如果要添加新的基于 `symfony/console` 的命令，只要把它添加到应用中即可。

## 定义命令

要创建新的命令，可以扩展 `Symfony\Component\Console\Command\Command` 或者 `Spiral\Console\Command` 类，后者在前者的基础上提供了一些方便的语法糖。

```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    const NAME        = 'my:command';
    const DESCRIPTION = 'This is my command';

    const ATTRIBUTES = [];
    const OPTIONS    = [];

    /**
     * 执行命令
     */
    protected function perform()
    {
    }
}
```

通过常量 `NAME` 和 `DESCRIPTION` 来提供命令的名称和描述。定义完之后，即可在命令行下调用这个命令：

```bash
$ php app.php my:command 
```

## 参数和选项

Spiral 的 `Command` 类让定义参数和选项变得非常简单：

```php
const ARGUMENTS = [
    ['argument', InputArgument::REQUIRED, '参数说明。']
];
    
const OPTIONS = [
    ['option', 'c', InputOption::VALUE_NONE, '选项说明。']
];
```

要通过参数和/或选项读取用户输入，可以使用 `$this->argument("argName")` 以及 `$this->option("optName")`.

## 执行命令

 命令的具体执行过程放在 `perform` 方法中。`perfom` 方法也是支持 Spiral 的方法注入的，此外通过 `$this->input` 和 `$this->output` 属性可以处理输入和输出。

```php
protected function perform(MyService $service)
{
    $this->output->writeln($service->doSomething());
}
```

## 辅助方法

在继承自 `Spiral\Console\Command` 类的命令中，有很多辅助方法可以使用。下面给出的示例代码都可以在 `perform` 方法中使用：

单行输出：

```php
$this->writeln("hello world");
```

输出内容但不换行：

```php
$this->write("hello world");
```

格式化输出并且不换行：

```php
$this->sprintf("Hello, <comment>%s</comment>", $name);
```

> 这个方法的签名与 PHP 内置的 `sprintf` 方法一致。

检查当前命令被调用的信息详细模式是否高于 `OutputInterface::VERBOSITY_VERBOSE`（默认值）：

```php
dump($this->isVerbose());
```

> 在控制台命令中可以随意使用 `dump` 方法。

还可以使用输出表格：

```php
$data = [
    [1, 'foo'],
    [2, 'bar'],
    [3, 'baz'],
];

$table = $this->table([
    '第一列',
    '第二列',
]);

foreach ($data as $row)
{
    $table->addRow([
        $row[0],
        $row[1]
    ]);
}

$table->render();
```

会输出带颜色的：

```bash
+----------+----------+
| 第一列    | 第二列    |
+----------+----------+
| 1        | foo      |
| 2        | bar      |
| 3        | baz      |
+----------+----------+
```

还可以添加分隔行：

```php
$table->addRow(new \Symfony\Component\Console\Helper\TableSeparator());
```
